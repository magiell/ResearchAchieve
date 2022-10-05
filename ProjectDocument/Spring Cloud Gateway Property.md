## SCG의 프로퍼티 설정 가이드

### cors 설정
- all open으로 예시이고 제약사항의 경우 치환 하면 됨
```yml
# all open
globalcors:
  cors-configurations:
    '[/**]':
      allowed-origin-patterns: [ "*" ]
      allowed-origins: [ "*" ]
      allowed-headers: [ "*" ]
      allowed-methods:
        - POST
        - GET
        - PUT
        - DELETE
```

### default-filters 설정
- 디폴트 필터를 설정하면 라우팅 할때 마다 필터가 호출
- 클래스를 구현 뒤 클래스 이름을 명시하면 됨
- `AbstractGatewayFilterFactory`를 상속 구현 하면 됨
    ```java
    public class DefaultFilter extends AbstractGatewayFilterFactory<DefaultFilter.Config>
    ```

### routes 설정
- id, uri, predicates, filters 4개를 보통 설정
- id: 라우팅할 서비스의 아이디
- uri: 라우팅할 서비스 호스트
- predicates: 라우팅 조건 리스트
- filters: 라우팅 필터 리스트
- yml
    ```yaml
    - id: test
      uri: http://test.service:8080
      predicates:
        - name: Path
          args:
            patterns: /test/** # 해당 패턴으로 들어오는 요청
        - name: Method
          args:
            methods:
               - GET #GET Method만
      filters:
        - name: RewritePath
          args:
            regexp: /test/(?<path>.*) #해당 경로의 정규식 패스를
            replacement: /$\{path} #해당 경로 형식으로 재설정해주고 호출
    ```

### filters 설정
- 보통 많이 쓰는 필터
    1. 경로 재설정 `RewritePath`
    2. 헤더 재설정 `RewriteResponseHeader`
    3. 헤더 삭제
    `RemoveResponseHeader`
    4. 추가 헤더 설정 `AddRequestHeader`
- 결국 `AbstractGatewayFilterFactory`의 상속구현체들이니 있는 것은 Bean으로 등록 되어있으니 필요한 부분은 라이브러리에서 찾아보고 구현을 추천
- 위의 routes의 filters가 해당 설정 값이니 `AbstractGatewayFilterFactory` 구현체이므로 설정 값들은 비슷하게 값을 가져오려고 시도할 것