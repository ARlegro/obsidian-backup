
## Customize AutenticationEntryPoint
- is capable of handling 401 ERROR which is not authenticated access
- By default
	- LoginUrlAuthenticationEntryPoint is used for form-based login 
	- BasicAuthenticationEntrypoint is used for HTTP Basic 

> [!WARNING] 
> - **Case1. form-based** 
> 	- If i use login-form by security, there is `LoginUrlAuthenticationEntryPoint` 
> 	- ❗**don't have to customize it** which is specific UI flow
> 	- Cuz it is not with the JSON response
> 	  
> - **Case2. REST APIs or non-UI flows** 
> 	- most scenario is based on HTTP Flow
> 	- which **want to return a custom error msg or json body** instead of a redirce or browser popup

#### 0. `commence()` : Target of cumizing method 
✅**When this method invoked** 
- commence() is called when **authentication is requrired but the user is not authenticated**

✅**Role** 
- *Build the HTTP response*  (for unauthenticated access)
- *Decide what to send back* (ex. status, headers, body etc)
- *Replace the default behavior* such as redirecting to login page or showing a blowser

#### Step 1. Customize header 
```java 
public class CustomBasicAuthenticationEntryPoint implements AuthenticationEntryPoint {  
    @Override  
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {  
    response.setHeader("custom-error-reason", "Authentication failed");  
    response.sendError(HttpStatus.UNAUTHORIZED.value(), HttpStatus.UNAUTHORIZED.getReasonPhrase());  
}  
        
```
setHeader(key, value)

setError
- Instead of 401, I can throw any status ex. 403, 500 

#### Step2. Configure entrypoint
>[!tip] There is 2 method of configure entry point 

**1‍⃣ Cofigure in HttpBasic( )** 
> 💢It's only for HttpBasic() 💢
> So, is going to be executed or considered during the httpbasic flow (**Only Login Flow**)
```java 
http.formLogin(flc -> flc.disable());  
http.httpBasic((hbc) -> 
			hbc.authenticationEntryPoint(new CustomBasicAuthenticationEntryPoint())); // Invoke
```
customize httpBasic()
- By withDefault() ➡ register BasicAuthenticationEntryPoint for error handling 
- Need to customize for Error customizing 

**2‍⃣ Cofigure in exceptionHandling()** ⭐⭐ << It's Globa❗❗
> It is a global setting

```java 
http.httpBasic(hbc ->  
        hbc.authenticationEntryPoint(new CustomBasicAuthenticationEntryPoint())); 
http.exceptionHandling(ehc ->  
        ehc.authenticationEntryPoint(new CustomBasicAuthenticationEntryPoint()));
```

#### Step 3. Customize body 
```java 
public class CustomBasicAuthenticationEntryPoint implements AuthenticationEntryPoint {  
    @Override  
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {  
        LocalDateTime currnetTimeStamp = LocalDateTime.now();  
        String path = request.getRequestURI();  
        String messgae = (authException!=null && authException.getMessage()!=null)  
                ? authException.getMessage()  
                : "Unauthorized access.";  
        response.setHeader("custom-error-reason", "Authentication failed");  
        response.setContentType(MediaType.APPLICATION_JSON_VALUE + ";charset=UTF-8");  
  
        String jsonResponse =  
                String.format("{\"timestamp\": \"%s\", \"status\": %d, \"error\": \"%s\", \"message\": \"%s\", \"path\": \"%s\"}",  
                        currnetTimeStamp, HttpStatus.UNAUTHORIZED.value(), HttpStatus.UNAUTHORIZED.getReasonPhrase(), messgae, path);  
				
        //response.sendError(HttpStatus.UNAUTHORIZED.value(), HttpStatus.UNAUTHORIZED.getReasonPhrase());  <<< This is delegatin error handling to Braowser 
        
        response.getWriter().write(jsonResponse);
```
setConentType : "application/json;charse=UTF-8"

Construct the JSON response
- `response.getWriter().write(jsonResponse)`


