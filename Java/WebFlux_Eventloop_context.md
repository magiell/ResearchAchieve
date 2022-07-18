## Webflux Security - Armeria 적용
------
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

    