
In Spring securty, there is no need to worry about all kind of Exception 
>[!Tip] It's only going to worry and handle security related Exception

### Types of Security Exception 
There are 2 types of Exception related Security
#### 1. AuthenticationException 
- **401** status : is not authenticated 
- ex. BadCredential, UsernameNotFound, InsufficientAuthentication
- is Handled by AuthenticationEntryPoint(Interface)

#### 2. AccessDeniedException
- **403** status : forbidden error
- When try to **invalid or incorrect PATH** 



### How to handle Exception

>[!QUESTION] ❓ How to handle these types of Exception ➡ **ExceptionTranslationFilter**
>- If i want to customize exception result ➡ Override ExceptionHandler provided by default 

#### Spring's Exception Handler that provided 

![[supporter/image/PDF 10.png]]

Each of Exception mentioned invoke each Interface which is responsible handle Ex 
1. `AuthenticatonEx` invoke ➡ `AutenticationEntryPoint`✅
	- default **Implementation** of `AutenticationEntryPoint` : `BasicAuthenticationEntryPoint`
	  
2. `AccessDeniedException` invoke  ➡ `AccessDeniedHandler`✅
	- By default implementation : 
	- When 

So, To handle Exception in detail ➡ need to override Interfaces (~Entrypoint, ~handler)

구현체 들어가보면 아래와 같이 되어 있음

```java
public class ExceptionTranslationFilter ....{

private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {  
    try {  
        chain.doFilter(request, response);  
    } catch (IOException ex) {  
        throw ex;  
    } catch (Exception var8) {  
        Throwable[] causeChain = this.throwableAnalyzer.determineCauseChain(var8);  
        RuntimeException securityException = (AuthenticationException)this.throwableAnalyzer.getFirstThrowableOfType(AuthenticationException.class, causeChain);  
        if (securityException == null) {  
            securityException = (AccessDeniedException)this.throwableAnalyzer.getFirstThrowableOfType(AccessDeniedException.class, causeChain);  
        }  
  
        if (securityException == null) {  
            this.rethrow(var8);  
        }  
  
        if (response.isCommitted()) {  
            throw new ServletException("Unable to handle the Spring Security Exception because the response is already committed.", var8);  
        }  
			  // This is Security Exception 
        this.handleSpringSecurityException(request, response, chain, securityException);  
    }
```


```java 
private void handleSpringSecurityException(HttpServletRequest request, HttpServletResponse response, FilterChain chain, RuntimeException exception) throws IOException, ServletException {  
    if (exception instanceof AuthenticationException) {  
        this.handleAuthenticationException
				        (request, response, chain, (AuthenticationException)exception);  
    } else if (exception instanceof AccessDeniedException) {  
        this.handleAccessDeniedException
				        (request, response, chain, (AccessDeniedException)exception);  
    }
```
➡ The type of exception determines which handler will process it
➡ Of course, **Even thoght handleAccessDeniedException is happen, Exception can  be changed to AuthenticationException in progrees**


--- 
[[401 - Customize AutenticationEntryPoint]]
[[403 - Customize AccessDeniedHandler]]
