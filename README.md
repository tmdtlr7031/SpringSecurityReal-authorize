# 스프링시큐리티 실전편
- (3) 사용자 DB등록 및 PasswordEncoder
  - Spring Security 5 부터 평문 암호화르 위해 `PasswordEncoderFactories.createDelegatingPasswordEncoder();`를 이용하여 빠르게 구현 가능 (`DelegatingPasswordEncoder`이용함)
  - 하지만 default 값이 `bcrypt`를 이용하여 인코딩을 하게 된다. 그럼 sha256으로 바꾸고싶다면..?
  - SecurityConfig.java에서 `new StandardPasswordEncoder()`이렇게 작성하면 되지만 @Deprecated 되었다. 안에 들어가 확인해보니
  - `DelegatingPasswordEncoder`구현체르 사용하도로 가이드 되어 있다.
  - 즉, 기존에 암호화가 안되어 있다며 `bcrypt`를 따르고, 기존에 다른 암호화 방식으로 되어 있는 형식 (ex. {sha256}asjdkajsnkjc)들을 마이그레이셔 작업할떄는 유용할 것.. 
  - 물론 평문 Or 해당 포맷이 아닌 경우에 `DelegatingPasswordEncoder`을 이용하면 Exception날 것으로 예상된다.
  - 참고
   > https://stackoverflow.com/questions/65796088/how-override-the-default-bcryptpasswordencoder-created-through-passwordencoderfa
   > https://www.inflearn.com/questions/349010
 
```java
// DelegatingPasswordEncoder.java

private PasswordEncoder defaultPasswordEncoderForMatches = new UnmappedIdPasswordEncoder();

@Override
public boolean matches(CharSequence rawPassword, String prefixEncodedPassword) {
	if (rawPassword == null && prefixEncodedPassword == null) {
		return true;
	}
	String id = extractId(prefixEncodedPassword);
	PasswordEncoder delegate = this.idToPasswordEncoder.get(id);
	if (delegate == null) {
		return this.defaultPasswordEncoderForMatches.matches(rawPassword, prefixEncodedPassword);
	}
	String encodedPassword = extractEncodedPassword(prefixEncodedPassword);
	return delegate.matches(rawPassword, encodedPassword);
}
```
 
```java
// UnmappedIdPasswordEncoder.java

// Default {@link PasswordEncoder} that throws an exception telling that a suitable
// {@link PasswordEncoder} for the id could not be found.
private class UnmappedIdPasswordEncoder implements PasswordEncoder {

	@Override
	public String encode(CharSequence rawPassword) {
		throw new UnsupportedOperationException("encode is not supported");
	}

	@Override
	public boolean matches(CharSequence rawPassword, String prefixEncodedPassword) {
		String id = extractId(prefixEncodedPassword);
		throw new IllegalArgumentException("There is no PasswordEncoder mapped for the id \"" + id + "\"");
	}
}
```

---


- (4) DB 연동 인증 처리(1) - CustomUserDetailsService
  - 기존에 메모리 방식은 테스트용으로 사용 가능할 것 같다.
  - UserDetails 인터페이스를 위해 User 클래스를 상속받는 AccountContext 클래스 생성
  - 스프링 시큐리티가 최종적으로 UserDetailsService르 이용하여 인증처리를 한다.
  - 여기엔 UserDetails가 필요하다. 그래서 UserDetails를 구현한 User 클래스를 상속받아 커스텀한 AccountContext를 만들게 된 것이다.

  ```java
   @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 메모리 방식 인증처리
        String password = passwordEncoder().encode("1111");
        auth.inMemoryAuthentication().withUser("user").password(password).roles("USER","MANAGER","ADMIN");
        auth.inMemoryAuthentication().withUser("manager").password(password).roles("MANAGER","ADMIN");
        auth.inMemoryAuthentication().withUser("admin").password(password).roles("ADMIN");

        // DB 정보를 통한 인증처리
        auth.userDetailsService(customUserDetailsService);
    }
  ```

- (5) DB 연동 인증 처리(2) - CustomAuthenticationProvider
  - 첫번째 생성자는 AuthenticationManager에게 전달할 때 인증필터가 해당 생성자를 통해 사용자 정보를 넘기게 된다.
  - 반면 두번째 생성자는 인증에 대한 검증이 완료된 경우 사용한다.
  ```java
 	 public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
		super(null);
		this.principal = principal;
		this.credentials = credentials;
		setAuthenticated(false);
	}

	public UsernamePasswordAuthenticationToken(Object principal, Object credentials,
			Collection<? extends GrantedAuthority> authorities) {
		super(authorities);
		this.principal = principal;
		this.credentials = credentials;
		super.setAuthenticated(true); // must use super, as we override
	}
  ```
  
  
  - 문제발생
   - CustomAuthenticationProvider를 작성하고 SecurityConfig애서 DI를 통해서 사용하려 했지만 순환참조 오류가 발생했다. (생성자 주입의 중요성!!)
   - `SecurityConfig`에서 `customAuthenticationProvider`를 DI 하기 위해 `customAuthenticationProvider`를 @Component를 통해 빈으로 등록하지만 `customAuthenticationProvider`에서 `passwordEncoder`를 DI하기 위해 `SecurityConfig`를 바라보기 때문이었다.
   - @Component를 빼고 `SecurityConfig`에서 별도로 빈을 만들자


  ```java
    @Component
    @RequiredArgsConstructor
    public class CustomAuthenticationProvider implements AuthenticationProvider {

    private final UserDetailsService customUserDetailsService;
    private final PasswordEncoder passwordEncoder;

    /**
     * 인증에 대한 검증 부분
     * - authentication : 사용자가 입력한 아이디, pw 등이 담겨있음
     * - id, pw 외 다른 검증도 추가할 수 있다.
     */
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
	String username = authentication.getName();
	String password = (String) authentication.getCredentials();

	AccountContext accountContext = (AccountContext) customUserDetailsService.loadUserByUsername(username);

	// 입력한 패스워드와 DB상의 패스워드 일치 여부
	if (!passwordEncoder.matches(password, accountContext.getPassword())) {
	    throw new BadCredentialsException("BadCredentialsException");
	}

	// 최종적으로 인증에 성공한 정보를 담아 Provider를 호출한 AuthenticationProcessingFilter에게 반환
	UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(accountContext.getAccount(), null, accountContext.getAuthorities());
	return authenticationToken;
    }

    /**
     * 파라미터로 넘어온 authentication이 UsernamePasswordAuthenticationToken과 일치하면
     * CustomAuthenticationProvider가 인증을 처리한다.
     */
    @Override
    public boolean supports(Class<?> authentication) {
	return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
  }
  
  ```
  ```java
    @EnableWebSecurity
    @RequiredArgsConstructor
    public class SecurityConfig extends WebSecurityConfigurerAdapter {

      private final UserDetailsService customUserDetailsService;
      private final AuthenticationProvider customAuthenticationProvider;

      @Override
      protected void configure(AuthenticationManagerBuilder auth) throws Exception {
          auth.authenticationProvider(customAuthenticationProvider);
      }
      
      @Bean
      public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
      }
    
    // ------------------------ 변경 후
    // CustomAuthenticationProvider.java => @Component 제거
    
    // SecurityConfig.java
    // private final AuthenticationProvider customAuthenticationProvider; 제거
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(authenticationProvider());
    }
      
    @Bean
    public AuthenticationProvider authenticationProvider() {
        return new CustomAuthenticationProvider(customUserDetailsService, passwordEncoder());
    }
    
  ```
  - 기존 방식과의 차이점?
    - `auth.userDetailsService(customUserDetailsService);` 
      - DaoAuthenticationProvider가 customUserDetailsService를 이용
      - 인증을 시도하는 사용자의 정보가 DB에 존재하는지를 검증하는 비즈니스 로직을 담당
    - `auth.authenticationProvider(authenticationProvider());` 
      - CustomAuthenticationProvider를 통해 customUserDetailsService를 이용
      - 실제 사용자의 인증 처리를 위한  비즈니스 로직을 구현하는 역할 (customUserDetailsService보다 넓은 개념)
   - 인증 검증(ex.패스워드 검증)같이 특별한 과정이 필요하다면 Provider를 이용하고 그게 아니라면 DaoAuthenticationProvider만 간단하게 사용해도 무방하다. 아니면 둘 다 사용해도 되고..


---

 - 로그아웃 및 화면 보안처리
   - LogoutFilter 이용
     - 해당 필터 적용되는 조건 
       - HTTP Method POST, 요청 URL이 `/logout`인 경우가 해당 됨
       - 예외
         - `http.csrf().disable()` 설정한 경우에는 GET 방식도 LogoutFilter가 처리한다.

   - `http.csrf().disable()` 설정 없이 GET 요청인 경우 ? -> `SecurityContextLogoutHandler` (LogoutFilter도 결국은 이 핸들러 사용한다.)
     - LoginController의 /logout 메서드 참고
   
   - 인증 여부에 따른 메뉴 노출 처리
   - 타임리프에서는 `sec:authorize="isAnonymous()"` 이런식으로 쓰고 JSP에서는 아래처럼 쓰는 것 같다. 이는 필요할 때 찾아보자.
     ```html
     <sec:authorize access="isAuthenticated()">
       <li class="nav-item"><a class="nav-link text-light" href="<c:url value="/logout"/>">로그아웃</a></li>
     </sec:authorize>
     ```
   
     - sec태그 이용과 기능을 이용하기 위해선 의존성추가 + 타임리프에 선언이 필요하다.
       ```java
       // 타임리프 상단에 추가
       xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity5" 추가
       
       // 의존성(gradle 기준)
       implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity5:3.0.4.RELEASE'
       ```

  - 인증 부가 기능 (WebAuthenticationDetails, AuthenticationDetailsSource)
    - 사용자 request에 의해 AuthenticationFilter가 Authentication 객체르 만들고 여기의 details에 해당하는 `WebAuthenticationDetails`를 `AuthenticationDetailsSource`가 생성한다. 
    - WebAuthenticationDetails에느 기본적으로 remoteAddress, SessionId가 있고 그 외에 뷰에서 넘겨주느 추가적이 파라미터를 받을 수 있다.
      ```html
      <!-- login.html -->
      <form th:action="@{/login_proc}" class="form-signin" method="post">
          <input th:type="hidden" th:value="secret" name="secret_key">
              <div class="form-group">
                  <input type="text" class="form-control" name="username" placeholder="아이디" required="required" autofocus="autofocus">
              </div>
              <div class="form-group">
                  <input type="password" class="form-control" name="password" placeholder="비밀번호" required="required">
              </div>
              <button type="submit" class="btn btn-lg btn-primary btn-block">로그인</button>
       </form>
      ```
      ```java
      public class FormWebAuthenticationDetails extends WebAuthenticationDetails {

	  private String secretKey;

	    // 사용자가 전달하는 추가적인 파라미터들을 저장하는 역할
	    public FormWebAuthenticationDetails(HttpServletRequest request) {
	        super(request);
	        secretKey = request.getParameter("secret_key");
	    }

	    public String getSecretKey() {
	        return secretKey;
	    }
      }
	  
      // CustomAuthenticationProvider 에서 검증처리 할 수 있다.
      // secretKey 검증
      FormWebAuthenticationDetails details = (FormWebAuthenticationDetails) authentication.getDetails();
      String secretKey = details.getSecretKey();
      if (!"secret".equals(secretKey)) {
          throw new InsufficientAuthenticationException("InsufficientAuthenticationException");
      }
      ```
    - `FormAuthenticationDetailsSource` 와 `FormWebAuthenticationDetails` 파일 참고
    
----

  - 인증 성공 / 실패 핸들러 커스텀
    - 각각 `SimpleUrlAuthenticationSuccessHandler`, `SimpleUrlAuthenticationFailureHandler` 를 상속받는다.
    - `CustomAuthenticationSuccessHandler`
      - `RequestCache` 와 `RedirectStrategy`를 이용한다.
        - RequestCache의 기본 구현체는 `HttpSessionRequestCache`로 미인증 이용자가 접근했던 정보르 담고있는 `SavedRequest`를 세션에 저장하는 역할을 한다.
        - 참고로 SavedRequest의 기본 구현체는 `DefaultSavedRequest`이며 Null인 경우도 존재한다. (ex. 인증 전 다른 자원 접근 -> 인증 예외 발생 -> 로그인페이지 = savedRequest 없음)
        - RedirectStrategy를 통해 인증 성공 시 디폴트 경로 설정과 특정 경로로 sendRedirect할 수 있다.
          - 단, SecurityConfig의 defaultSuccessUrl이 있으면 설정할 필요 없다.
          - 추가로 SecurityConfig에 defaultSuccessUrl 설정 시 successHandler 보다 위에 선언해야 한다.
            - API 설정이 아래에 위치할 수록 위에 위치한 설정을 덮어쓰게 된다. 따라서 defaultSuccessUrl이 successHandler보다 아래에 있으면 제대로 동작 안함

    - `CustomAuthenticationFailureHandler`
      - `setDefaultFailureUrl("/login?error=true&exception="+errorMessage);` 이렇게 쿼리 파라미터로 정보 넘기는데 Spring의 BindingResult에서 받아줄 수 있는지는 안해봐서 모르겠다고 한다. 나중에 테스트 해보는것도.. 
      - 추가로 스프링 시큐리티에서 "/login?error=true&exception="+errorMessage 를 "/login"으로 인식하지 않고 문자열 전체를 경로로 인식한다
      - 따라서 SecurityConfig에 PermitAll 추가해야한다. -> `.antMatchers("/login*").permitAll()`
      - 인증 실패의 경우 `super.onAuthenticationFailure(request, response, exception);`를 사용하는데 사실 없어도 동작한다. 
        - 하지만 실패 핸들러쪽의 경우 부모쪽으로 위임하는게 여러모로 편리하여 해당 코드를 사용한다.

----

 ### [ 비동기 챕터 ] 
  - 2) 인증 필터 - AjaxAuthenticationFilter
  ```java
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
  ```
  
  위 소스에서는 굳이 빈으로 안 만들었다. 강사님의 강의 스타일인가 싶었는데 누군가 질의응답을 남겨서 기록한다.
  > `AuthenticationManager`는 스프링 시큐리티에서 초기화 시 생성하게 된다. 빈이 아니라 일반 객체로!
  > 스프링 시큐리티는 `HttpSecurity`에 있는 `SharedObject`를 가지고 여기에 객체들을 넣어놓고 참조하는 식으로 운용한다. 
  > -> 스프링 빈이 아님 또한 Config에서 빈으로 만드는 건 여러 위치에 DI하기 위함이다.


---

  - 3) 인증처리자 - AjaxAuthenticationFilter ~ 4) 인증 핸들러 - AjaxAuthenticationSuccessHandler, AjaxAuthenticationFailureHandler
    - 전체적인 흐름 익숙해지자..!
      - 필터 -> 토큰 생성(사용자가 입력한 정보 기반으로 미인증상태임) -> 인증매니저에게 위임 -> Provider에게 처리 위임 -> 처리 후 토큰 반환 (인증된 토큰) -> ...
    - 트러블슈팅
      - 상황
        - 폼방식과 Ajax방식(=REST용 서버통신) SecurityConfig 파일로 분리하고 Ajax방식을 테스트를 하던 중 ajax방식임에도 불구하고 폼방식에서 설정한 successHandler가 적용되는 상황
        - 강의처럼 302 상태코드가 나와야하지만 그렇지 않았다.
      - 원인
        - DI 방식을 필드 주입으로 진행하는 강의와는 다르게 생성자 주입으로 하고 있었고 `AuthenticationSuccessHandler`, `AuthenticationFailureHandler`를 구현하고 있었다.
        - 이때 구현한 내 파일에서 @Component를 선언하여 빈으로 생성했고 `SecurityConfig`에서 주입하고 있었다.
          ```java
		  @Order(1)
		  @EnableWebSecurity
		  @RequiredArgsConstructor
		  public class SecurityConfig extends WebSecurityConfigurerAdapter {
		    private final AuthenticationSuccessHandler customAuthenticationSuccessHandler;
		    private final AuthenticationFailureHandler customAuthenticationFailureHandler;
		  }
          ```

        - 즉, ajax용 성공,실패 핸들러를 빈으로 만들어 `AjaxSecurityConfig`에 DI 하지 않는 이상 인터페이스 `AuthenticationSuccessHandler`, `AuthenticationFailureHandler`는 폼 방식에서 사용하는 핸들러를 주입받아 적용하게 된 것이었다.
      - 해결
        - `AjaxAuthenticationSuccessHandler`, `AjaxAuthenticationFailureHandler`를 생성하고 둘 다 @Component를 통해 빈으로 생성
        - AjaxSecurityConfig에서 DI 하고 테스트 -> 의도한대로 정상 동작함













