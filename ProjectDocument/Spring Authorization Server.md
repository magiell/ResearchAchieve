
## OIDC Provider 및 인가 서버 구축

### 요구 사항 설계
-  자체 로그인 서버 - 인증 (소셜 로그인 X)
-  JWT
- Legacy 유저 테이블 호환
- Java 17 도입
- 커스텀 인증 페이로드 호환 (암호화 페이로드 -> 복호화)

### 기본 세팅
-  Security Filter Setup
```java

@Bean  
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
    OAuth2AuthorizationServerConfigurer configurer = authorizationServerConfigurer();  
  
    http  
            .csrf(AbstractHttpConfigurer::disable)  
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))  
            .authorizeHttpRequests(authorize -> authorize  
                    .requestMatchers("/login", "/login_submit").permitAll()  
                    .anyRequest().authenticated()  
            )            .exceptionHandling(ex -> ex.authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/login")))  
            .formLogin(AbstractHttpConfigurer::disable)  
            .httpBasic(AbstractHttpConfigurer::disable)  
            .anonymous(AbstractHttpConfigurer::disable)  
            .securityContext(httpSecuritySecurityContextConfigurer -> httpSecuritySecurityContextConfigurer  
                    .securityContextRepository(new HttpSessionSecurityContextRepository()))  
            .authenticationProvider(authenticationProvider())  
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))  
            .logout(Customizer.withDefaults());  
  
    // 추가 configurer 적용  
    http.apply(this.customAuthenticationConfigurer);  
    http.apply(configurer);  
  
  
    configurer  
            .oidc(Customizer.withDefaults())  
            .authorizationService(this.authorizationService)  
            .registeredClientRepository(this.registeredClientRepository);  
  
    return http.build();  
}

```
-  Custom Filter
```java
public class CustomAuthenticationFilter extends UsernamePasswordAuthenticationFilter {  
  
    ...
  
    @Override  
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException  
    {  
	    // `HttpServletRequestWrapper`를 통해 리퀘스트 객체를 변경 후 인증
        return super.attemptAuthentication(changeCustomRequestWrapper(request), response);  
    }
}

```


