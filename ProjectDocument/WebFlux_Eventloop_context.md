## Webflux Security - Armeria 삽질 히스토리

### Dependency
- **Armeria 1.7.0**
- **Spring Webflux 2.7.1**
- **Spring Security 5.7.2**

### Configuration
<details>
  <summary><b>ServerConfigure.class</b>
  </summary>

```java
@Configuration
public class ServerConfigure {

    @Bean
    public static EventLoopGroup eventLoopGroup() {
        // armeria eventloop 설정
        return EventLoopGroups.newEventLoopGroup(Flags.numCommonWorkers(), "stem-webflux-worker");
    }

    @Bean
    public static BlockingTaskExecutor blockingTaskExecutor() {
        // armeria blocking worker 설정
        return BlockingTaskExecutor.builder()
                .threadNamePrefix("stem-webflux-blocking-thread")
                .numThreads(Flags.numCommonBlockingTaskThreads()).build();
    }

    @Bean
    public ArmeriaServerConfigurator armeriaServerConfigurator() {
        return serverBuilder -> {
            // 이벤트 루프 스레드 지정
            serverBuilder.workerGroup(eventLoopGroup(), true)
                    .startStopExecutor(CommonPools.workerGroup());

            serverBuilder.blockingTaskExecutor(blockingTaskExecutor().unwrap(), true);

            serverBuilder.requestIdGenerator(RequestId::random);
            serverBuilder.decorator(LoggingService.newDecorator());

            // 60초 타임아웃
            serverBuilder.idleTimeout(Duration.ofSeconds(60));
            serverBuilder.requestTimeout(Duration.ofSeconds(60));

            serverBuilder.decorator(EncodingService.newDecorator());
            serverBuilder.decorator(ContentPreviewingService.newDecorator(Integer.MAX_VALUE, StandardCharsets.UTF_8));
            serverBuilder.decorator(DecodingService.newDecorator());

            serverBuilder.decorator(CorsService.builderForAnyOrigin().newDecorator());

            //노출 제한 헤더 off
            serverBuilder.disableServerHeader();
            serverBuilder.disableDateHeader();
        };
    }

    @Bean
    public DocServiceConfigurator docServiceConfigurator() {
        return docServiceBuilder -> {
            // 웹 클라이언트 설정
        };
    }
}
```      
</details>

### Example Server
- **SSE(Server Sent Event) Server**
- **Simple Authentication & Authorization**

### Issue History
1. **SSE Server reference Deprecated**
    - EmitterProcessor deprecated로 인한 Sinks 대체
2. **RequestTimeout**
    - 활성화 SSE Stream의 타임아웃 문제
    - Ping 전용 Stream 필요
3. **Webflux Security**
    - security filter를 거치면 이벤트 루프 스레드가 바뀐다?
    - 커스텀 인증 & 인가 시스템 적용 (Cookie-token)


### Find Solution
1. Deprecated class 교체 작업
    <details>
      <summary>
        Used <a href="https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Sinks.html">Sinks</a> Class
      </summary>

      ```java
      /*
      * 1. 브로드캐스트 지원 (구독자 모두에게 notice message가 가야함)
      * 2. SSE는 구독 시점 이전의 내용은 받을 필요 X
      */
      this.multicastStream = Sinks.many().multicast().directAllOrNothing();
      ```

    </details>
2. Request Timeout
    <details>
      <summary>
        <a href="https://github.com/filipeGuerreiro/armeria/blob/739991a3f222eb029e82815aa5aced3f1d4d2605/core/src/main/java/com/linecorp/armeria/common/Flags.java#L623">Default request timeout (Armeria) </a>
      </summary>

      ```java
      /*
      * 1. Ping Stream 추가
      * 2. Armeria의 RequestTimeout(60s) 안에 Ping 메시지를 보내고 응답이 있으면 timeout을 60초 연장 시켜준다
      * 3. 클라이언트 연결이 끊기면 doOncancel 호출
      */
      private Flux<ServerSentEvent<String>> pingStream() {
        return Flux.interval(Duration.ofSeconds(30))
                .publishOn(Schedulers.fromExecutor(ServiceRequestContext.current().eventLoop()))
                .map(idx -> {
                    ServiceRequestContext.current().setRequestTimeout(TimeoutMode.EXTEND, Duration.ofSeconds(60));
                    return idx;
                })
                .map(i -> ServerSentEvent.builder("PING")
                        .id("NaN")
                        .event("ping")
                        .comment("keep-alive").build())
                .doOnCancel(() -> {
                    log.info("client is disconnected");
                });
    }
      ```

    </details>
    <details>
      <summary>
        <a href="https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Sinks.html">mergeWith</a> Stream
      </summary>

      ```java      
      /*
      * 1. 스트림에서 filter통해 condition이 참 인 데이터를 받을 수 있다 (관심 데이터만)
      * 2. 즉 구독한 client의 식별자를 알 수 있다면 Unicast도 가능하지 않을까?
      */
      public Flux<ServerSentEvent<String>> subscribeSSEChannel(String clientChannels) {
        return getSSEFlux()
                .filter(Objects::nonNull)
                .filter(sse -> requireNonNull(sse.id()).equalsIgnoreCase(clientChannels) || sse.id().equalsIgnoreCase("NaN"));
    }

    /*
    * SSE의 stream에 PingStream을 merge
    */
    private Flux<ServerSentEvent<String>> getSSEFlux() {
        return multicastStream.asFlux()
                .publishOn(Schedulers.fromExecutor(ServiceRequestContext.current().eventLoop()))
                .mergeWith(pingStream())
                .log();
    }
      ```

    </details>
3. Security 적용
    <details>
      <summary>
        Webfilter (<a href="https://github.com/spring-projects/spring-security/issues/9200">request cache</a>) issue
      </summary>

      ```java
      /*
      * 1. 시큐리티에서 리퀘스트 캐시 옵션을 disable 하지 않으면 디폴트(in-memory cache)가 만들어짐
      * 2. 내부적으로 디폴트 필터를 만들때 가장 큰 문제가 생기는데 리액터의 스케쥴러 루프가 새로 생성 되어 버림
      * 3. Armeria-webflux의 이벤트 루프가 갑자기 소실 되어버려서 기껏 만들어놨던 컨텍스트들이 유실 되어버림 (armeria request)
      * 4. 현재(5.x) 버전에서 이슈는 open상태이고 disable 시켜서 webfilter chain에 포함 시키지 않으면 이벤트 루프 유지시키는 방향으로
      */
      httpSecurity.addFilterBefore((exchange, chain) -> {
            // 필터 적용하기 전까지 armeria eventloop 
            log.info(Thread.currentThread().toString());
            return chain.filter(exchange);
        }, SecurityWebFiltersOrder.SERVER_REQUEST_CACHE);

      httpSecurity.addFilterAfter((exchange, chain) -> {
            // 필터를 거치면 parallel(???) 로 바뀜
            log.info(Thread.currentThread().toString());
            return chain.filter(exchange);
        }, SecurityWebFiltersOrder.SERVER_REQUEST_CACHE);
      ```

    </details>
    <details>
      <summary>
        Context <a href="https://engineering.linecorp.com/ko/blog/hello-armeria-bye-spring/">Propagation</a>
      </summary>

      - 기존 Spring MVC는 Context 정보들은 ThreadLocal로 관리하는데 1 요청당 1개의 스레드를 사용하기 때문에 컨텍스트 유지가 가능
      - Netty기반 프레임워크는 eventloop 스레드가 다수의 요청을 처리하는 방식이기 때문에 Threadlocal 방식의 Context들을 처리가 불가능
      - Request 처리가 완료 될 때까지 이벤트루프 스레드의 스위칭이 일어나도 Context를 유지하는 방법이 필요함
      - 앞서 설계 아이디어가 리퀘스트 처리 할 때 컨텍스트를 가지고 있는 이벤트 루프를 계속 프로세스 처리 하도록 요청시키면 되지 않을까란 생각을 가지고 있으나 외부 변수나 파악하지 못한 라이브러리 로직에 의해 스레드가 임의로 변경 될 수 있다는 걸 확인
      - 스레드 스위칭을 컨트롤 하기엔 너무 변수가 많기 때문에 각 컨텍스트 정보를 Copy-On-Write 하는 방식으로 전파하는 방법을 사용
      - 기존 mvc security 말고 webflux용 security는 기본적으로 `ReactiveSecurityContextHolder`로 전파된 security context를 전역 접근 할 수 있다.
      
      > <details>
      > <summary>기본 원리</summary>
      >
      >```java
      >String key = "message";
      >Mono<String> r = Mono.just("Hello")
      >  .flatMap(s -> Mono.deferContextual(ctx ->
      >   Mono.just(s + " " + ctx.get(key))))
      >.contextWrite(ctx -> ctx.put(key, "World"));
      >```
      ></details>

      > <details>
      > <summary> Armeria Context Propagation Utility</summary>
      >
      >```java
      >RequestContextHooks.enable();
      >```
      ></details>
      

    </details>
    
    <details>
      <summary>
        특이한 레거시 인증 서비스
      </summary>

      * Oauth2 access token을 쿠키로 보관함 
      * 그래서 항상 인증 전에 쿠키를 조회해서 토큰으로 돌려놔야하는 작업이 필요함(converter)
      * authentication process를 손수 API를 다 때려서 인증 정보를 만드는 작업
      * 인증 정보를 모으려면 API를 두번 때려야 함 (공통 계정 정보 + 사용 서비스 계정 추가 정보)
      * 결국 모든 것을 수동으로 구현 ㅠㅠ

    </details>
    
