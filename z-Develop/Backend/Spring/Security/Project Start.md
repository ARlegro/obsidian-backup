
### REST API 
![[image/PDF.png]]


### See SecurityConfiguration
> for customize, need to know default SecurityConfiguration




**✔How to find
- search :  Ctrl + N ➡ `SpringBootWebSecurityConfiguration`

**✔Core Structure (simplified)**
```java
@ConditionalOnDefaultWebSecurity  
static class SecurityFilterChainConfiguration {  
    SecurityFilterChainConfiguration() {  
    }  
  
    @Bean  
    @Order(2147483642)  
    SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {  
        http.authorizeHttpRequests((requests) -> ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)requests.anyRequest()).authenticated());  
        http.formLogin(Customizer.withDefaults());  
        http.httpBasic(Customizer.withDefaults());  
        return (SecurityFilterChain)http.build();  
    }
```
자동 설정에 의해 추가되는 기본 filterChain들을 설정하는 Bean 
보면 인증, 인가, 폼 로그인, http 기본 등의 설정을 셋팅
<br>

🔐Explanation
1. `http.authorizeHttpRequest( () -> ...)`
	- **Secure all incoming HTTP requests**
	- All requests require authentication.
	- Anonymouse people can't access APIs

2. `http.formLogin(Customizer.withDefaults());`
	- **Enable form-based login**  
	- Automatically **provides a default login pages** at `/login`
	- Provides the following behavior 
		- **Redirects** to `login?error` when fail to login 
		- **Redirects** to the original requested page when success to login
		- **Enables session-based authentication.**

 3. `http.httpBasic(Customizer.withDefaults());`
	 - **Enables HTTP Basic authentication**
	 - Credentials are sent in the Authorization header
	 - used for RESTful APIs or command-line toold

> [!Example ]
> - `formLogin()`
> 	- **웹 페이지 기반 UI 로그인**
> 	- 브라우저용 로그인 폼 제공
> - `httpBasic()`
> 	- **REST 클라이언트나 API 도구(Postman 등)를 위한 인증** 
> 	- HTTP 요청 헤더를 통한 인증 지원 (REST용)


>[!tip] For Customize, Override defaultSecurityFilterChain
 



