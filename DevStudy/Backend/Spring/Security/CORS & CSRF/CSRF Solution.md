## Solution to handle CSRF attacks 

### 💢Problem (Root Cause)
- Backend ⭐ **don'have a mechanism to identify** whether given request coming is from own UI application or from hacker's website 
	(CSRF 공격의 본질 = 브라우저의 자동 쿠키 전송을 악용하는 것)
- Cuz **Both website(original + malicious) has same JSESSIONID cookie** , both send automatically this cokkie 

### ✅Solution - CSRF token
> **With the CSRF token**, Server **can differentiate** whether the requests is comming from original website or not 

#### Concept 
> Backend Server pass CSRF tocken to client for differentiating  when sending JSESSIONID with 
- CSRF = secure random token
- **Prevent** CSRF attacks 
- **Unique** per user session (각 사용자 세션마다 고유하게 발급)
- **Large Random value** Cuz to make it difficult to guess 
- ❓When client recieve this token
	- Sent after login from server


#### How it works ❓
1. Server generates and sends the CSRF token (usually after login).
2. Client (browser or SPA) includes the CSRF token in all **state-changing** requests (e.g., POST, PUT).
	- The **Client should include this token** in each reqeust header or payload 
	- CSRF has to send as part of the cookie
3. **Server validates the token** before processing the request.

#### Why it works ❓
>[!tip] Using Character ➡ JS cannot acess other origin tocken/cookie
- Due to the **"Same-Origin Policy"❗** : Hacker cannot read CSRF tocken,  
- **Same-Origin Policy** is : 
	- Hacker cannot read CSRF in JS When exist CSRF token 
	- Cuz JS cannot read the CSRF tocken from another origin 
	- (**CSRF 토큰이 있는 환경에서는** 공격자가 JS로 위조 요청은 해도, **토큰을 제대로 포함시킬 수 없다**)
- So, **Hacker will not able to send cookie value** inside the requestHeader or requestPayload 

### Summary 
- CSRF = sending a fake rquest using user's cookie 
- CSRF tocken = additional verification to prevent CSRF 
- **JS cannot read the CSRF tocken from another origin** ⭐⭐⭐

--- 

## Impl in Spring Security

> [!QUESTION] Why Cookie기반의 Csrf를 추천할까? (Session에도 CsrfToken을 담을 수 있지만 비추)
> 1. **서버 부하가 덜하다 (stateless)**
> 	- Session 방식은 서버 메모리에 세션 정보를 저장하므로 상태유지가 필요 
> 	- Cookie에 방식은 토큰을 브라우저에 저장하고 client가 헤더에 싣는 방식
> 	- 서버는 token만 비교하면 되므로 **상태 유지 불필요** 
> 	<br> 	  
> 2. **SPA와 REST API에 최적** 
> 	- SPA 구조에서는 Cookie + 헤더 방식이 표준처럼 사용된다.
> 	- Spring Security 가 session기반으로 만든거는 전통적인 MVC를 바탕으로 만들었기 때문이다.
> 	- 현재 대부분은 Front, back end 분리되어있으므로 Cookie 기반이 더 적합하다.


### Core Component 
#### 1. CsrfToken - Interface
```java 
public interface CsrfToken extends Serializable {  
  String getHeaderName();  
  
  String getParameterName();  
  
  String getToken();
```
- Provides the information about an expected CSRF token.
- headername 
- parameterName : the name when sending form 
- token = random value 

#### 2. DefaultCsrfToken - Impl
```java 
public final class DefaultCsrfToken implements CsrfToken {  
  
  @Serial  
  private static final long serialVersionUID = 6552658053267913685L;  
  
  private final String token;  
  private final String parameterName;  
  private final String headerName;  
  
  /**  
   * Creates a new instance   * @param headerName the HTTP header name to use  
   * @param parameterName the HTTP parameter name to use  
   * @param token the value of the token (i.e. expected value of the HTTP parameter of  
   * parametername).   */  
	public DefaultCsrfToken(String headerName, String parameterName, String token) {  
		   Assert.hasLength(headerName, "headerName cannot be null or empty");  
		   Assert.hasLength(parameterName, "parameterName cannot be null or empty");  
		   Assert.hasLength(token, "token cannot be null or empty");  
		   this.headerName = headerName;  
		   this.parameterName = parameterName;  
		   this.token = token;  
  }  
  
  @Override  
  public String getHeaderName() {  
   return this.headerName;  
  }  
  
  @Override  
  public String getParameterName() {  
   return this.parameterName;  
  }  
  
  @Override  
  public String getToken() {  
   return this.token;  
  }
```



#### 3. Token Repo - Interface 
>Developer cannot generate token direct during the login. 
>Instead, **CsrfTokenRepo generate it**

**☑Role** 
- **Genereate** token 
- **Store** : store token to validate wheter request has same token or not 
- **Load** : to differentiate 
```java
public interface CsrfTokenRepository {
		
		CsrfToken generateToken(HttpServletRequest request);
		
		void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response);

		CsrfToken loadToken(HttpServletRequest request);
		
		default DeferredCsrfToken loadDeferredToken(HttpServletRequest request, HttpServletResponse response) {  
		  return new RepositoryDeferredCsrfToken(this, request, response);  
		}
```
CookieCsrfTokenRepository has may methods like generate()


#### 4. Token Repo - Impl
```java 
/**A CsrfTokenRepository that persists the CSRF token in a cookie named "XSRF-TOKEN" and reads from the header "X-XSRF-TOKEN" following the conventions of AngularJS. When using with AngularJS be sure to use withHttpOnlyFalse()
**/
public final class CookieCsrfTokenRepository implements CsrfTokenRepository {
```
- Saves CSRF token in a cookie (e.g., `XSRF-TOKEN`) and expects it in headers (`X-XSRF-TOKEN`)

참고로 이 구현체는 `withHttpOnlyFalse()`라는 특이한 메서드가 있다.
```java
public static CookieCsrfTokenRepository withHttpOnlyFalse() {  
  CookieCsrfTokenRepository result = new CookieCsrfTokenRepository();  
  result.cookieHttpOnly = false;  
  return result;  
}
```
- `HttpOnly = false` 옵션은 
	- **JS에서 쿠키를 읽을 수 있도록 허용**.


### CsrfFilter : ❓How CSRF Filter works 
> Validate CsrfToken 

**순서 정리** 
1. 토큰 로드 (loadDeferredToken)
2. 요청이 CSRF 보호 대상인지 확인
3. 요청의 헤더/파라미터에서 받은 토큰 값을 추출
4. 저장된 토큰과 비교
5. 불일치 시 AccessDeniedException 발생
   
```java 
public final class CsrfFilter extends OncePerRequestFilter {

		@Override  
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)  
	   throws ServletException, IOException {  
	  
	  DeferredCsrfToken deferredCsrfToken = this.tokenRepository.loadDeferredToken(request, response);  
	  request.setAttribute(DeferredCsrfToken.class.getName(), deferredCsrfToken);  
	  this.requestHandler.handle(request, response, deferredCsrfToken::get);  
	  
	  if (!this.requireCsrfProtectionMatcher.matches(request)) {  
	   if (this.logger.isTraceEnabled()) {  
	    this.logger.trace("Did not protect against CSRF since request did not match "  
	      + this.requireCsrfProtectionMatcher);  
	   }  
	   filterChain.doFilter(request, response);  
	   return;  
	  }  
	  
	  CsrfToken csrfToken = deferredCsrfToken.get();  
	  String actualToken = this.requestHandler.resolveCsrfTokenValue(request, csrfToken);  
	  
	  if (!equalsConstantTime(csrfToken.getToken(), actualToken)) {  
	   boolean missingToken = deferredCsrfToken.isGenerated();  
	   this.logger  
	    .debug(LogMessage.of(() -> "Invalid CSRF token found for " + UrlUtils.buildFullRequestUrl(request)));  
	   AccessDeniedException exception = (!missingToken) ? new InvalidCsrfTokenException(csrfToken, actualToken)  
	     : new MissingCsrfTokenException(actualToken);  
	   this.accessDeniedHandler.handle(request, response, exception);  
	   return;  
	  }  
	  
	  filterChain.doFilter(request, response);  
}
```

--- 
## CSRF Configure 

> [!INFO]  By default = Session base 
> - Default Using `HttpSessionCsrfTokenRepository`
> 	- Spring Security **use impl based session** Cuz traditional webApp
> 	- But nowdays, Fronend use SPA(react, angular). These framework **fit to CSRF token based Cookie**
> - So, Token is **Saved based session** 
> - Validate 
> 	- 기본적으로 위험한 메서드만 검증 
> 	- ✅ POST/PUT/PATCH/DELETE 등 
> 	- ❌ GET/HEAD/TRACE/OPTIONS 
```java 
public final class CsrfConfigurer<H extends HttpSecurityBuilder<H>> extends AbstractHttpConfigurer<CsrfConfigurer<H>, H> {  
	  private CsrfTokenRepository csrfTokenRepository = new HttpSessionCsrfTokenRepository();
```


### Configuring CSRF
> I'll use token based cookie. (default = based session)

#### Step 
✅**Step1. Change Token Repo**
- default is session 
- So, need to change token repo based cookie

✅**Step2. Set `withHttpOnlyFalse()`**
- This setting **is letting** the UI Framework to **read Cookie value** 
- Note : This method do setting automatically `CookieCsrfTokenRepo` 
	(이 with~ 메서드는 내부적으로 자동으로 쿠키기반 repo를 생성하고 반환한다)
- 💢**If `true`** 
	- Cookie can be only read by the browser 
	- Then, **JS-React code cannot attach this cookie** 

>[!tip] 쿠키 기반 저장소 설정 코드 
```java 

http. 
		.csrf(csrfConfig ->
					csrfConfig.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))

```
```java 
public final class CookieCsrfTokenRepository implements CsrfTokenRepository {
  
		public static CookieCsrfTokenRepository withHttpOnlyFalse() {  
		  CookieCsrfTokenRepository result = new CookieCsrfTokenRepository();  
		  result.cookieHttpOnly = false;   // 자동으로 false 
		  return result;  
		}
```
## Problem of general token generation 

**💢일반적인 CSRF 토큰 생성 방식과 문제** 
Spring Security는 기본적으로 **모든 요청마다 세션을 생성**하고, **해당 세션에 CSRF 토큰을 저장**합니다.
- 하지만 GET 요청처럼 **검증이 필요 없는 요청**에도 매번 토큰을 생성하면 **불필요한 세션 낭비**가 생김.
- 또한, **Stateless API 서버**에서는 세션 자체도 쓰지 않기에 이런 방식은 부적절.
 
 > [!WARNING] ❗문제 정리 ❗
 > 1. 불필요한 세션 생성
 > 2. 서버 메모리 낭비
 > 3. 성능 저하 



### Solution : Lazy generate 

#### What is Lazy Generate❓
- It doesn't generate CSRF Tocken when first request
- Instead, After invoke `request.getAttribute(CsrfToken.class.getName())` or `csrfToken.getToken()`, it generate token !!
    > 즉, **필요할 때만(실제로 꺼내쓸 때만) 만든다** = Lazy Generation
	```java 
	CsrfToken csrfToken = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
	String token = csrfToken.getToken(); // 이 시점에서 토큰 생성됨!
	```
- It makes performance good

>[!EXAMPLE] Why it works 
>- 불필요한 토큰 생성 방지
>- Stateless 환경에 적합
>- 불필요한 CSRF 필터 동작 최소화 


#### 해결책 : Using OncePerRequestFilter to Lazy generate 

**❓`OncePerRequestFilter`란?**
- It's one of Spring filters
- 한 요청(request) 당 **딱 한 번만 실행되는 필터**를 만들고 싶을 때 사용
- 기본 `Filter`는 여러 번 실행될 수도 있기 때문에 안정성 측면에서 이걸 상속함.

❓Why need it 
- SPA/ REST 초기 요청에서는 Spring 템플릿 엔진이 관여하지 않아 토큰을 꺼내 쓸 코드가 서버에 없다.
- **서버-사이드 뷰 레이어가 없는** 상황에서도 `_csrf.getToken()`을 **의도적으로 호출**해 줌으로써 첫 인증 이후 응답에 `XSRF-TOKEN` 쿠키를 보낼 수 있다.
<br>
✅Step1. OncePerRequestFilter 오버라이딩 
```java 
public class CsrfCookieFilter extends OncePerRequestFilter {  
  
  @Override  
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,  
      FilterChain filterChain) throws ServletException, IOException {  
  
    CsrfToken csrfToken = (CsrfToken) request.getAttribute(CsrfToken.class.getName());  
    // Render the token value to a cookie by causing the differed token to be loaded   
csrfToken.getToken();  
    filterChain.doFilter(request, response);  
  }
```

✅Step 2. addFilterAfter 
- `BasicAuthenticationFilter` 이후에 실행되도록 필터 체인에 삽입
- 보통 CSRF 토큰은 Authentication filter 이후에 생성·삽입하면 충분함
	- 인증이 완료된 후 토큰 생성해서 클라이언트에 제공하기 위함 
```java
.csrf( ... )
.addFilterAfter(new CsrfCookieFilter(), BasicAuthenticationFilter.class)
```

>[!tip] **세션 생성**에 드는 비용이 Authentication 비용보다 더 나가기 때문에 Authentication 먼저 처리하는게 비용적 측면에서도 유리 
> 핵심은 '**세션 사용 여부**'
> CPU 연산은 거의 비슷.
> 단지, **메모리 및 네트워크 자원 절감 목적의 최적화** 


📘**Conclusion**
- `OncePerRequestFilter` : 요청당 1번 실행 보장되는 필터 
- addFilterAfter( ~ , ~) : 인증 이후 인증에 대한 응답 전에 Token cookie를 클라이언트에 주기 위해 

#### 번외 :  커스텀 없이, 언제 CsrfFilter가 일어날까?
> 아래 사진을 보면 알 수 있듯이, 
> 기본적으로 CsrfFilter는 **`CsrfFilter`는 기본적으로 HeaderWriterFilter 바로 뒤, LogoutFilter 앞**에 자동 배치된다.
> 즉, Authentication이 되기 전에 일어난다. 
> 인증 실패(401) 상황에서는 굳이 쿠키를 줄 필요가 없음
https://medium.com/%40AlexanderObregon/the-mechanics-of-spring-boots-security-filter-chain-1fe716ba22db
![[Pasted image 20250529110243.png]]
## CsrfTokenRequestAttributeHandler

- 로그인 후 첫 POST 요청이 들어오면 `CsrfFilter`가 동작
- 그런데 이 필터는 토큰을 어디서 **어떻게 읽어야 할지** 알아야 함
- 그 역할을 담당하는 게 바로 → `CsrfTokenRequestAttributeHandler`

### 전체 흐름 
1. 로그인 요청 (POST /login)
	- 인증 성공 시 서버는 JSESSIONID 쿠키 생성 
	- 추가로, CSRF 토큰도 같이 내려준다 (쿠키 기반일 경우 XSRF-TOKEN 쿠키)
	  
2. 이후 클라이언트가 재요청 
	- 브라우저가 자동으로 request에 XSRF-TOKEN 쿠키를 붙인다.
		- 헤더명 = X-XSRF-TOKEN
		- 쿠키명 =  XSRF-TOKEN

### Problem 
> [!WARNING] 여기서 문제 :  CSRF Filter가 토큰을 받고 요청 헤더를 읽는 기능이 없음 
> - 어떻게 헤더로부터 추출할지를 알아야되는데, 기본값은 그런 로직이 없다
```java
public final class CsrfFilter extends OncePerRequestFilter {
		@Override  
		protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)  
		   throws ServletException, IOException {

```
- CsrfFilter는 기본적으로 Spring Security 내부에서 CSRF 검증 처리 (서버 저장값 vs 요청 토큰값 비교)
- doFilterInternal 메서드가 **request로부터 CSRF 토큰을 extract하는 것을 도와주기 위해 config가 필요**하다
	- Cuz 기본적으로 `CookieCsrfTokenRepository`는 토큰을 **쿠키로 내려주지만**, 요청 헤더에서 읽는 방식은 자동이 아님.
	- 따라서, 요청에서 토큰을 **헤더나 파라미터에서 꺼내주는 핸들러**를 명시해줘야 `CsrfFilter`가 올바르게 토큰 비교를 수행할 수 있음


### Solution : 핸들러 구현
>  CsrfTokenRequestAttributeHandler를 구현하면 된다.
https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html#csrf-token-request-handler-custom
>[!INFO] 공식 문서 
>The `CsrfTokenRequestAttributeHandler` makes the `CsrfToken` available as an `HttpServletRequest` attribute called `_csrf`
>(Note : `_csrf` 라는 속성 이름을 바꿀 수도 있다)
- 이 구현체는 request header or parameter로부터 token value를 추출한다
- By default, 추출 방법
	- 헤더 : `X-CSRF-TOKEN` or `X-XSRF-TOKEN`이라는 request 헤더로 
	- 파라미터 : `_csrf`

By default handler
- 기본적인 핸들러는 BREACH 공격 방지 기능이 설정되어 있다.
- 따라서, 아래와 같이 CSRF 공격에 맞는 Handler로 변경이 필요하다. ⭐

✅**Config**
```java
public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
		// 핸들러 
    CsrfTokenRequestAttributeHandler handler = new CsrfTokenRequestAttributeHandler();

    http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()) // 쿠키로 내려주고
            .csrfTokenRequestHandler(handler) // 요청에서 꺼내주는 방법 설정
        )
```

✅**CsrfTokenRequestAttributeHandler**
```java
public class CsrfTokenRequestAttributeHandler implements CsrfTokenRequestHandler {  
  
  private String csrfRequestAttributeName = "_csrf";

	@Override  
public void handle(HttpServletRequest request, HttpServletResponse response,  
   Supplier<CsrfToken> deferredCsrfToken) {  
  ...  
  CsrfToken csrfToken = new SupplierCsrfToken(deferredCsrfToken);  
  request.setAttribute(CsrfToken.class.getName(), csrfToken);
```

CsrfTokenRequestAttributeHandler역할
1. 요청(Request)에서 **CSRF 토큰을 추출**한다
2. 꺼낸 토큰을 Request Attribute에 심어준다. 

  
  >이전에는 BREACH 공격 방지 전용 핸들러였다면, 이 설정을 통해 CSRF방지 전용 핸들러로 바뀌게 되었다 
