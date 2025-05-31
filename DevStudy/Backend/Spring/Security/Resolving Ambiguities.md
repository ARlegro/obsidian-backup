
>[!QUESTION] 헷갈렸던 것 
>1. 세션이 남아있는 로그인 시도 시 Authentication객체를 안 만들고 바로 authN 과정을 겪지 않는다는데, 그럼 어디서 자세히 일어나는걸까?
>2. `Securitycontextholder` vs `Basicauthenticationfilter` vs `AbstractAuthenticationProcessingFilter`


### 참고 : filter 순위 
```
2 SecurityContextHolderFilter  : SecurityContext의 저장 및 정리

....
40 AbstractPreAuthenticatedProcessingFilter
42 OAuth2LoginAuthenticationFilter   ← AbstractAuthenticationProcessingFilter 하위
44 UsernamePasswordAuthenticationFilter ← AbstractAuthenticationProcessingFilter 하위
49 BearerTokenAuthenticationFilter
50 BasicAuthenticationFilter         ← HTTP Basic
52 RequestCacheAwareFilter
56 AnonymousAuthenticationFilter
58 SessionManagementFilter
59 ExceptionTranslationFilter
60 FilterSecurityInterceptor         ← 인가(Authorization)
```


| Filter                                                                                                                                                          | Added by                             |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| [CsrfFilter](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html)                                                                       | `HttpSecurity#csrf`                  |
| [BasicAuthenticationFilter](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/basic.html)                                       | `HttpSecurity#httpBasic`             |
| [UsernamePasswordAuthenticationFilter](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/form.html#servlet-authentication-form) | `HttpSecurity#formLogin`             |
| [AuthorizationFilter](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html)                                      | `HttpSecurity#authorizeHttpRequests` |
| OAuth2LoginAuthenticationFilter                                                                                                                                 | `HttpSecurity#oauth2Login`           |

`formLogin()`을 호출하지 않으면 `UsernamePasswordAuthenticationFilter` 자체가 체인에 없다

---
## 세션있는 요청 시 어디서 처리?

현재 알고 싶은 내용은, 이미 인증된 사용자의 경우(세션 기반) 불필요한 인증 과정을 거치지 않도록 security가 어떻게 처리하는지이다.
### SecurityContextHolderFilter
SecurityContextHolderFilter는 전체 filter의 2번째 순위일 정도로 거의 먼저 동작한다.

**✅역할** 
- `SecurityContext` 로드 및 `ThreadLocal` 저장: (뒤에서 자세히 다루니 참고)
- ThreadLocal 클리어 

✅**초기 로그인 시** Flow based Session 인증 
1. 사용자가 로그인 정보 제출
2. `SecurityContextHolderFilter`는 `HttpSession`에 `SecurityContext`가 없음을 확인하고 `ThreadLocal`에 아무것도 올리지 않는다.
3. 인증 필터들은 Authenti 없음을 확인하고 인증 메서드 실행
	- 예시 : AbstractAuthenticationProcessingFilter
	```java
	private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {  
			// 인증이 필요한지 확인 ⭐⭐⭐
			if (!this.requiresAuthentication(request, response)) {  
					chain.doFilter(request, response);  
					
			} else {  
					try {  
							// 인증이 필요할 경우 attemptAuthentication 실행 ⭐⭐⭐
							Authentication authenticationResult = this.attemptAuthentication(request, response);  
							if (authenticationResult == null) {  
									return;  
	```
4. 인증 성공 시 성공 정보를 담은 Authentication 객체를 `SecurityContextHolder`에 저장
5. 동시에 `SecurityContextRepository`를 통해 이 `SecurityContext`가 `HttpSession`에 저장
   (클라는 재요청시 httpsession을 보내면 시큐리티가 확인하는 구조 )

**✅이후 요청 (세션 유효)** ⭐⭐⭐⭐
1. **클라이언트 요청** : Browser는 자동으로 SessionID(JSSESSIONID)를 포함하여 요청
2. **WAS의 세션 매핑** : HttpSession 객체 찾기
	- WAS는 요청 헤더에 포함된 `JSESSIONID` 값을 읽어 들인다.
	- WAS는 자신의 메모리(또는 세션을 저장하는 다른 저장소)에 보관하고 있는 `HttpSession` 객체들 중에서 **해당 `JSESSIONI`에 일치하는 객체를 찾아낸다.**
3. **2번 성공 시** 
	- WAS가 `JSESSIONID`에 해당하는 `HttpSession` 객체를 성공적으로 찾아냈다면, **`HttpSession` 객체를 현재 요청(`HttpServletRequest`)과 연결시켜 준다.**
	- `SecurityContextHolderFilter`는 요청에 포함된 세션 ID를 사용하여 자신의 메모리에 저장된 session정보 중 id가 일치하는 HttpSession을 찾아온다.
4. **SecurityContextHolderFilter 등장** : 인증 정보 로드 
	- request객체로부터 HttpSession객체를 찾아본다
		- HttpSessionSecurityContextRepository가 `SPRING_SECURITY_CONTEXT`라는 특정 이름으로 저장되어 있는 속성을 활용해 찾아본다.
		```java
		public class HttpSessionSecurityContextRepository implements SecurityContextRepository {  
		    public static final String SPRING_SECURITY_CONTEXT_KEY = "SPRING_SECURITY_CONTEXT";
		```
	- 3번 과정을 통해 성공적으로 저장되어 있다면 4번에서 **찾은 HttpSession 객체에는 Authentication객체를 담은 SecurityContext가 있을 것**이다.
	  
5. 다른 인증 필터들의 조건부 실행 : 불필요한 재인증 방지
	- 이후 순서대로 실행되는 각 인증 필터들은 `SecurityContextHolder.getContext().getAuthentication()`을 확인하고, 이미 `Authentication` 객체가 존재하기 때문에 자신의 `attemptAuthentication()` **로직을 건너뛰고 다음 필터로 넘어간다.**
6. 결과적으로 인증 과정 없이 곧바로 인가 및 요청 처리 로직 진행

> 그니까 세션 찾는거는 SecurityContextHolderFilter 등장 전에 WAS에서 처리하는거네

>[!tip] HttpSession 정의
>- 서버 측 객체
>- 웹 서버(WAS)가 클라이언트와의 상태를 유지하기 위해 사용하는 객체
>- 각 클라마다 고유한 HttpSession 객체가 서버 메모리에 생성 
>- Security에서 SecurityContext를 HttpSession내부에 저장하기도 한다.

---
## Securitycontextholder` vs `Basicauthenticationfilter` vs `AbstractAuthenticationProcessingFilter`비교 

### 1. SecurityContextHolder
>역할 : SecurityContext의 저장 및 정리 

#### 관련 Concpet
**❓SecurityContext란❓**
- 현재 인증된 사용자의 정보(principal)와 권한(authorities)을 담고 있는 핵심 객체 
- 주로, `Authenticaiton` 객체 형태로 `SecurityContext` 내에 저장된다.

**❓SecurityContextHolder란❓**
- `SecurityContext`를 저장하고 접근할 수 있는 정적(static) 홀더 클래스
- 현재 스레드(Thread)와 관련된 `SecurityContext`를 저장하여 애플리케이션의 어느 코드에서든 인증 정보에 쉽게 접근할 수 있도록

#### Filter 코드 
 ```java 
 private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {  

		// 1. 필터 중복 적용 방지 
    if (request.getAttribute(FILTER_APPLIED) != null) {  
        chain.doFilter(request, response);  
    } else {  
		    // 필터가 적용되었음을 표시
        request.setAttribute(FILTER_APPLIED, Boolean.TRUE);  

				// 2. SecurityContext 지연 로드 (Deferred Loading)
        Supplier<SecurityContext> deferredContext = this.securityContextRepository.loadDeferredContext(request);  
  
        try {              
            // 3. SecurityContextHolder에 SecurityContext 설정
            // 이렇게 하면 현재 스레드 어디에서든 SecurityContext에 접근할 수 있다
					this.securityContextHolderStrategy.setDeferredContext(deferredContext);  
						// 4. 다음 필터 또는 서블릿으로 요청 전달
            chain.doFilter(request, response);  
        } finally {  
		        // 5. SecurityContext 정리 (필수!)
		        // 이는 다음 요청이나 다른 스레드에 이전 요청의 인증 정보가 남아있는 것을 방지
            this.securityContextHolderStrategy.clearContext();  
            // 6. 필터 적용 표시 제거
            request.removeAttribute(FILTER_APPLIED);  
        }
```

#### ClearContext의 중요성 (왜 정리?) ⭐⭐
>ThreadLocal의 특성

SecurityContextHolder는 기본적으로 `ThreadLocal`을 사용하여 `SecurityContext`를 저장한다. `ThreadLocal` 변수는 현재 실행 중인 스레드에 고유한 데이터를 저장한다.

여기서, **사전 지식**으로 알아야 할 점은 WAS는 성능을 위해 Thread Pool을 사용하며, **요청 처리가 끝나면** Thread를 **재사용**한다는 것이다(김영한 자바 편에서 배운 내용)

따라서, SecurityContext를 명시적으로 정리하지 않으면, 다음과 같은 문제가 발생 
1. 보안 취약 : 다음 요청이 이전 사용자의 인증 정보로 처리될 수 있는 문제
2. 잘못된 권한 부여 : 1번과 비슷
3. 메모리 누수 : `SecurityContext`객체가 계속 Thread에 묶여 있어 메모리 누수 

> 각 요청의 격리성을 보장하고, 보안 취약점을 방지하며, 자원을 효율적으로 관리하는 데 필수적인 작업

### 2. AbstractAuthenticationProcessingFilter
`AbstractAuthenticationProcessingFilter` 자체는 **추상 클래스**다.
실제 인증 흐름은 하위 필터들안의 `doFilter()` ➡ `attemptAuthentication()`에서 일어남 
**Impl은 딱 2개❗**
- UsernamePasswordAuthenticationFilter
	- Note : formLogin 방식이 disable이라면 이 filter는 없다. 💢
- WebAuthnAuthenticationFilter

### 3. BasicAuthenticationFilter

> Spring Security의 httpBasic 기능을 활성화 할 시 나타난다. 
> 순서 : 50

자세한 내용 : [Basic Authentication :: Spring Security](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/basic.html)
![[Pasted image 20250531104343.png]]
`Authorization: Basic <base64(id:pw)>` 헤더를 읽어 `AuthenticationManager.authenticate()` 수행 후, 성공하면 `SecurityContextHolder`에 저장

#### 절차  (레퍼런스 Copy)
1. When the user submits their username and password, the `BasicAuthenticationFilter` creates a `UsernamePasswordAuthenticationToken`, which is a type of `Authentication by extracting the username and password from the `HttpServletRequest`.

2. Next, the `UsernamePasswordAuthenticationToken` is passed into the `AuthenticationManager` to be authenticated. The details of what `AuthenticationManager` looks like depend on how the user information is stored.
3. If authentication fails, then _Failure_. 💢
	- The SecurityContextHolder is cleared out.
	- `RememberMeServices.loginFail` is invoked. If remember me is not configured, this is a no-op. See the [`RememberMeServices`](https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/web/authentication/RememberMeServices.html) interface in the Javadoc.
	- `AuthenticationEntryPoint` is invoked to trigger the WWW-Authenticate to be sent again. See the [`AuthenticationEntryPoint`](https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/web/AuthenticationEntryPoint.html) interface in the Javadoc.

4. If authentication is successful, then _Success_. ✅
	- The **Authentication is set on the SecurityContextHolder**.
	- `RememberMeServices.loginSuccess` is invoked. If rememberMe is not configured, this is a no-op. See the [`RememberMeServices`](https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/web/authentication/RememberMeServices.html) interface in the Javadoc.
	- The `BasicAuthenticationFilter` invokes `FilterChain.doFilter(request,response)` to continue with the rest of the application logic. 
    
**By default**, Spring Security’s HTTP **Basic Authentication support is enabled**. However, as soon as 
any servlet based configuration is provided, HTTP Basic must be explicitly provided.

#### ❓어떻게 username, password 읽을까❓

>[!QUESTION] 궁금했던게 만약, 프론트에서 입력하는게 3개라면 BasicAuthenticationFilter는 어떻게 처리할 수 있을까였다.

결론 먼저 말하자면, HttpBasic()이라고 해서 프론트가 JSON형식으로 데이터를 전송해주는게 아니였다.
프론트는 아래와 같은 예시로 user info를 헤더에 보낸다.
```javascript
const creds = btoa(userId + ':' + pwd);
fetch('/api/resource', {
  headers: { 'Authorization': 'Basic ' + creds }
});
```

✅Flow of Request by Basic Auth
1. **클라이언트**가 `Authorization: Basic ...` 헤더를 붙여 요청
2. 요청이 **FilterChain** 진입 → **`BasicAuthenticationFilter`**가 헤더 존재 여부 확인
3. 헤더가 있으면 `UsernamePasswordAuthenticationToken`을 만들어 **`AuthenticationManager`**로 전달
4. 인증 성공 시 `SecurityContextHolder`에 `Authentication` 저장, 실패 시 **`AuthenticationEntryPoint`**로 401 재도전 응답


> 즉, Basic Auth 방식은 Front에서 헤더로 정보를 담아 보낸거를 처리하는 방식
> 만약 Basic에 해당하는 헤더가 없다면 오류내겠지 


