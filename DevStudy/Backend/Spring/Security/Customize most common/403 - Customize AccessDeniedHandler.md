> 403 error related security = AccessDeniedException class
> - There are a lot of types of AccessDeniedException
> - ex. denied, service, csrf, invalidCsrfTocke, missingCsrfToken etc 
### Preview Exception and handler 

Exception 
```java 
public class AccessDeniedException  { 
    public AccessDeniedException(String file) {  
        super(file);  
    }  
  
    public AccessDeniedException(String file, String other, String reason) {  
        super(file, other, reason);  
    }
```


 For handling 403 error related security , need to customize `AccessDeniedHandler`' method 
```java 
public interface AccessDeniedHandler {  
    void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException;
```


### Customize AccessDeniedHandler



```java 
public class CustomAccessDeniedHandler implements AccessDeniedHandler {  
  
    @Override  
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {  
        LocalDateTime currentTimeStamp = LocalDateTime.now();  
        String path = request.getRequestURI();  
        String messgae = (accessDeniedException!=null && accessDeniedException.getMessage()!=null)  
                ? accessDeniedException.getMessage()  
                : "Authorization failed";  
        response.setHeader("custom-denied-reason", "Authorization failed");  
        response.setContentType(MediaType.APPLICATION_JSON_VALUE + ";charset=UTF-8");  
  
        String jsonResponse =  
                String.format("{\"timestamp\": \"%s\", \"status\": %d, \"error\": \"%s\", \"message\": \"%s\", \"path\": \"%s\"}",  
                        currentTimeStamp, HttpStatus.FORBIDDEN.value(), HttpStatus.FORBIDDEN.getReasonPhrase(), messgae, path);  
  
        response.getWriter().write(jsonResponse)
```

```java
public class ProjectSecurityProdConfig {  
  
    @Bean  
    SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
    ...
		http.exceptionHandling(ehc -> {  
		    ehc.authenticationEntryPoint(new CustomBasicAuthenticationEntryPoint());  
		    ehc.accessDeniedHandler(new CustomAccessDeniedHandler());  
});
```
1. Implement AccessDeniedHandler
2. And copy what i do when customize 401 ERROR [[401 - Customize AutenticationEntryPoint]]
3. Modify copied according to `AccessDenied`
	- EX. error status (401 -> 403), error name, variableName etc 

4. Register exceptionhandling 
	- This customized handler is not invoked httpbasic()
	- So, register to http.exceptionHandling()
	- When applications are getting AccessDeniedException, this handler will be executed and send back message to client 


>[!tip] There is accessDenied Option
>`ehc.accessDeniedHandler(new CustomAccessDeniedHandler()).accessDeniedPage("/denied")`
>- Whne 403 error is comming, the user will be redired to the path 
>- But this opition is **rarely used** Cuz most applications follow the REST API







