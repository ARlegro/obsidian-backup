
## BasicFilter

>[!SUCCESS]  This page will handle filters before step 2 
>- After this filter steps, **Authentication object is going to be created**
>- Authentication object ex. UsernamePasswordAuthenticationToken

![[PDF.pdf#page=13&rect=57,105,1398,647|Spring_Security, p.13]]


### DefaultLoginPageGeneratingFilter
```java
public DefaultLoginPageGeneratingFilter(UsernamePasswordAuthenticationFilter authFilter) {  
    this.loginPageUrl = "/login";  
    this.logoutSuccessUrl = "/login?logout";  
    this.failureUrl = "/login?error";  
    if (authFilter != null) {  
        this.initAuthFilter(authFilter);  
    }
```
 
✔Concept
- purpose : redirect the user to the login page By showing a page with username, pwd field
	 ![[Pasted image 20250518204446.png]]

✔main methods
```java
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {  
    boolean loginError = this.isErrorPage(request);  
    boolean logoutSuccess = this.isLogoutSuccess(request);  
    if (!this.isLoginUrlRequest(request) && !loginError && !logoutSuccess) {  
        chain.doFilter(request, response);  
    } else {  
        String loginPageHtml = this.generateLoginPageHtml
																		        (request, loginError, logoutSuccess);  
        
				response.setContentType("text/html;charset=UTF-8");             
				response.setContentLength(
													loginPageHtml.getBytes(StandardCharsets.UTF_8).length);
        response.getWriter().write(loginPageHtml);  
    }  
}
```
#generatingLoginPageHtml
- this methods receive "request, response, FilterChain"
- **Generate LoginPageHtml** 
- 
--- 
### AuthorizationFilter
```java
public class AuthorizationFilter extends GenericFilterBean {  
    private SecurityContextHolderStrategy securityContextHolderStrategy = SecurityContextHolder.getContextHolderStrategy();  
    private final AuthorizationManager<HttpServletRequest> authorizationManager;  
    private AuthorizationEventPublisher eventPublisher = new NoopAuthorizationEventPublisher();  
    private boolean observeOncePerRequest = false;  
    private boolean filterErrorDispatch = true;  
    private boolean filterAsyncDispatch = true;
```


✔Concept
- Responsible to **identify** whether a user is trying to access protected API or Path **without credentials** and **without authenticated session**

✔Code (dofilter method)
```java 
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {  
    if (!this.requestMatcher.matches(request)) {  
        if (this.logger.isTraceEnabled()) {  
            this.logger.trace("Did not match request to " + String.valueOf(this.requestMatcher));  
        }  
  
        filterChain.doFilter(request, response);  
    } else {  
        try {  
		        // attempt~ : process by implementation
            Authentication authenticationResult = this.attemptAuthentication(request, response);  
            if (authenticationResult == null) {  
                filterChain.doFilter(request, response);  
                return;  
            }  
  
            HttpSession session = request.getSession(false);  
            if (session != null) {  
                request.changeSessionId();  
            }  
  
            this.successfulAuthentication(request, response, filterChain, authenticationResult);  
        } catch (AuthenticationException ex) {  
            this.unsuccessfulAuthentication(request, response, ex);  
        }  
    }  
}
```
attemptAuthentication is defined inside **UsernamePasswordAuthenticationFilter**


---
## Filter When login - AbstractAuthenticationProcessingFilter
```java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean implements ApplicationEventPublisherAware, MessageSourceAware {
```
#Abstract-Class
### Concept and flow 
![[Pasted image 20250525153837.png]]


☑**Concept** of `AbstractAuthenticationProcessingFilter`
- **User 인증을 위한 기본 필터** : A base `Filter` **for authenticating a user’s credentials.** 
- **Authentication 객체 생성** 
	- User가 credentials을 submit ➡  `AbstractAuthenticationProcessingFilter` creates an `Authentication` from the `HttpServletRequest` to be authenticated
	- 이  filter의 구현체가 인증 객체를 만듬
	  
- **인증 객체 타입은 구현체에 따라 다르다** 
  The type of Authentication depends on implemenation of `AbstractAuthenticationProcessingFilter`
	- ex. `UsernamePasswordAuthenticationFilter` creates a `UsernamePasswordAuthenticationToken` from a _username_ and _password_ that are submitted in the `HttpServletRequest`.
	  
- **Have 2 implementation**
	1. WebAuthnAuthenticationFilter
	2. UsernamePasswordAuthenticationFilter

**❓What result of Authentication Fail or Success** 
1. ✅Case of Success
```java 
protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {  
    SecurityContext context = this.securityContextHolderStrategy.createEmptyContext();  
    context.setAuthentication(authResult);  
    this.securityContextHolderStrategy.setContext(context);  
    this.securityContextRepository.saveContext(context, request, response);  
    if (this.logger.isDebugEnabled()) {  
        this.logger.debug(LogMessage.format("Set SecurityContextHolder to %s", authResult));  
    }  
  
    this.rememberMeServices.loginSuccess(request, response, authResult);  
    if (this.eventPublisher != null) {  
        this.eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(authResult, this.getClass()));  
    }
```
- The Authentication is set on the `SecurityContextHolder
- `RememberMeServices.loginSuccess` is invoked
- etc
		  
2. ❌Case of Fail
```java 
protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response, AuthenticationException failed) throws IOException, ServletException {  
    this.securityContextHolderStrategy.clearContext();  
    this.logger.trace("Failed to process authentication request", failed);  
    this.logger.trace("Cleared SecurityContextHolder");  
    this.logger.trace("Handling authentication failure");  
    this.rememberMeServices.loginFail(request, response);  
    this.failureHandler.onAuthenticationFailure(request, response, failed);  
}
```
- The SecurityContextHolder is cleared out.
- `RememberMeServices.loginFail` is invoked.
- `AuthenticationFailureHandler` is invoked


### Code 

#### Overall
> 추상 클래스에 구현된 메서드 
- 인증이 필요한지 확인 
- 필요 시 : 인증 객체 생성 후 filter 메서드 invoke 
```java
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {  
		// Authentification이 필요한지 처리하는 if 문
    if (!this.requiresAuthentication(request, response)) {  
        chain.doFilter(request, response);  
    } else {  
        try {  
		        //attemptAuthentication -> do impletetion
            Authentication authenticationResult = this.attemptAuthentication(request, response);  
            if (authenticationResult == null) {  
                return;  
            }  
  
            this.sessionStrategy.onAuthentication(authenticationResult, request, response);  
            if (this.continueChainBeforeSuccessfulAuthentication) {  
                chain.doFilter(request, response);  
            }  
  
            this.successfulAuthentication(request, response, chain, authenticationResult);  
        } catch (InternalAuthenticationServiceException failed) {  
            this.logger.error("An internal error occurred while trying to authenticate the user.", failed);  
            this.unsuccessfulAuthentication(request, response, failed);  
        } catch (AuthenticationException ex) {  
            this.unsuccessfulAuthentication(request, response, ex);  
        }  
  
    }
	 
```

#### dofilter(..) : Core Logic Breadkdwon 
```java 
if (!this.**requiresAuthentication**(request, response)) {
    chain.doFilter(request, response);
    return;
}
```
- **Purpose** : Determines wheter the current **request should trigger authentication**

#### If authentication is required 
```java 
Authentication authenticationResult = this.attemptAuthentication(request, response);
```
- **Purpose** : Delegates the actual credential extraction and authentication attempt
- `attemptAuthentication` is logic provided by concrete filter like `UsernamePasswordAuthenticationFilter`
- Below is method in UsernamePasswordAuthenticationFilter
	1. Extract `usernaem` and `password`
	2. Build an Authentication token (UsernamePasswordAuthenticationToken)
		- **Authentication token is Authentication's child** ⭐
	3. Pass it to the Authentication
```java
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {  
    if (this.postOnly && !request.getMethod().equals("POST")) {  
        throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());  
    } else {  
        String username = this.obtainUsername(request);  
        username = username != null ? username.trim() : "";  
        String password = this.obtainPassword(request);  
        password = password != null ? password : "";  

				UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username, password);  
        this.setDetails(request, authRequest);  
        
        return this.getAuthenticationManager().authenticate(authRequest);  
    }
```

>[!Warning] AbstractAuthenticationProcessingFilter doesn't perform verification
>- Just, Determine if authentication is required
>- when is required, 
>	- **Delegate** extracition details(ex. UsernamePasswordAuthenticationToken) to other class  
>	- **pupulate details into security context** and **pass it  to AuthenticationManager**

>[!tip] security context is populated with Authentication details


Next = [[Flow2. Authentication Manager]]



