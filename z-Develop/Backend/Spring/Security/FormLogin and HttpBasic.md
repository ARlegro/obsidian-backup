

### Disable FormLogin

> In this section, i will diable formLogin page 

```java
http.formLogin((flc) -> flc.disable());
```

**❓Why formLogin().disable()**
- formLogin is spring Security's login page  `/login`
- Mostly, when i dev, i use REST API and front UI. 
- So there is no Login page made by Spring Security 

|                          | formLogin                       | REST API                                 |
| ------------------------ | ------------------------------- | ---------------------------------------- |
| Authentication UI        | Made by Spring Security`/login` | Made by front Developer (React, Vue)     |
| How to login             | form submit ➡ 세션 쿠키             | Invoke REST API (POST /login) + token 발급 |
| Result of Authentication | 리다이렉트 또는 세션 유지                  | Response with JSON + store token         |
| Flow security            | 서버 중심                           | 클라이언트 중심 (stateless)                     |


### HttpBasic()
#### When using only httpBasic (not loginForm)

☑If using the **httpBasic()** not loginForm 
- ❌UsernamePasswordAuthenticationFilter will not invoked  
- 💙Intead, **BasicAuthenticationFilter will invoked**
- it also conver request ➡ UsernamePasswordAuthenticationToken

```java
  
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {  
    try {  
        Authentication authRequest = this.authenticationConverter.convert(request);  
        if (authRequest == null) {  
            this.logger.trace("Did not process authentication request since failed to find username and password in Basic Authorization header");  
            chain.doFilter(request, response);  
            return;  
        }
    ....
    .....
```

![[Pasted image 20250519114513.png]]
- 위 사진은 BasicAuthenticationFilter의 doFilterInternal로직 중 this.authenticationConverter.convert(request); 이다.
- header를 보면 encoded value가 들어있음을 알 수 있다. (default : Base64)


### when diabling formLogin + httpBasic

```java
http.formLogin(flc -> flc.disable());  
http.httpBasic(hbc -> hbc.disable());
```

> a window like one shown below will appear 

![[Pasted image 20250519115241.png]]
☑ Error : 403 Forbidden 
- hbc.disable() : "HTTP Basic 인증창도 띄우지 말라"고 지시하는 것
- 인증되지도 않고, 인증할 방법도 없는 상태 




