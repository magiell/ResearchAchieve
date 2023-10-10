
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
-  Legacy User Integration
```java
@Component  
@Slf4j  
public class UserService implements UserDetailsManager {  

	// User Respository
    private final UserRepository userRepository;  
  
    public UserService(UserRepository userRepository) {  
        this.userRepository = userRepository;  
    }  
  
    @Override  
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {  
        var user = userRepository.findByAccount(email).orElseThrow();  
        log.info("account: {}", user.getAccount());  
        return CustomUserDetails.builder().user(user).build();  
    }
	
	...
}
```
- OIDC
```Java
OAuth2AuthorizationServerConfigurer configurer = authorizationServerConfigurer();

http.apply(configurer);

configurer  
		//enable oidc option
        .oidc(Customizer.withDefaults())  
        .authorizationService(this.authorizationService)  
        .registeredClientRepository(this.registeredClientRepository);
```

### 개발 이슈 히스토리
-  Form Login -> Custom Login 변경 후 인증 프로세스에서 인증 후 OIDC 리디렉션 과정에서 인증 컨텍스트가 유실되는 현상
-  `UserpasswordAuthenticationToken` 기반 인증에서 OIDC JWT에 SUB가 원하는 변수 값이 안 들어가는 현상 (UserInfo Mapping)
-  커스텀 인증 필터 적용 후 RequestMatcher 조건 상관없이 모든 리퀘스트에 필터가 적용되는 현상
-  인가 및 토큰 정보를 DB에 말아 넣거나 조회할 때 Jackson Mapper 이상 (Authority 객체)
-  OIDC UserInfo Url 접근 시에 인가서버의 Self ResourceServer 설정