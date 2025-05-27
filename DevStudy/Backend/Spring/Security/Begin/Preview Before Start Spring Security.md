

### 자동으로 세션을 기억

>[!QUESTION] 왜 한번 로그인 하면 그 이후부터는 물어보지 않게 됨?
>- 스프링 시큐리티는 내부적으로 로그인한 세션을 기억

#### 원리 
1. 로그인 성공 ➡ 인증 정보가 SecurityContextHolder에 저장
2. `SecurityContextPersistenceFilter`가 이를 **세션에 저장**
3. 이후 요청이 들어오면, 세션에 저장된 인증 정보를 다시 꺼내서 `SecurityContextHolder`에 넣음.
4. 따라서 사용자는 계속 로그인 상태를 유지함.

#### SecurityContextHolderFilter
> 참고 : deprecated 버전 : SecurityContextPersistenceFilter 
https://docs.spring.io/spring-security/reference/servlet/authentication/persistence.html#securitycontextholderfilter
![[Pasted image 20250525152305.png]]
- 인증 상태 유지의 핵심 역할 
- Responsible for loading the `SecurityContext` between requests using the `SecurityContextRepository`.
- **지연 로딩** : 세션 접근을 지연시키기 위해 `loadDeferredContext()` + Supplier 기반 구조 사용
	- 애플리케이션이 실행되기 전에, SecurityContextRepository로부터 `SecurityContext`를 얻어오고 SecurityContextHolder에 세팅한다
	- 하지만, SecurityContextHolderFilter은 load만 하는거지 SecurityContext를 아직 save하지 않은 것이다.

```java 
public class SecurityContextHolderFilter extends GenericFilterBean {
		...


		private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {  
    if (request.getAttribute(FILTER_APPLIED) != null) {  
        chain.doFilter(request, response);  
    } else {  
        request.setAttribute(FILTER_APPLIED, Boolean.TRUE);  
        // 지연 로딩 
        Supplier<SecurityContext> deferredContext = this.securityContextRepository.loadDeferredContext(request);  
  
        try {  
            this.securityContextHolderStrategy.setDeferredContext(deferredContext);  
            chain.doFilter(request, response);  // 인증/인가 
        } finally {  
            this.securityContextHolderStrategy.clearContext();  
            request.removeAttribute(FILTER_APPLIED);  
        }  
  
    }  
}

```
흐름 
1. 사용자 요청 
2. `SecurityContextHolderFilter`가 `loadDeferredContext()`로 Supplier를 등록 (지연 로딩)
3. dofilter 진행으로 인증/인가 실행 
4. 필터 체인 완료 후 
	- 필터 체인이 완료되면 `SecurityContextHolderFilter`가 다시 등장해 `clearContext()`로 ThreadLocal을 비운다.

>[!QUESTION] ❓언제 Context가 생성되는가?
>아래 상황중 하나라도 발생하면 생성 
>1. `UsernamePasswordAuthenticationFilter`가 인증 여부 확인
>2. `@PreAuthorize`, `hasRole()`, `getPrincipal()` 등 인증 정보 참조
>3. 커스텀 필터나 Controller에서 `SecurityContextHolder.getContext()` 호출
>> 이 시점에 supplier.get()이 호출된다, 

```scss
[요청 시작]
   ↓
SecurityContextHolderFilter
   ↓
loadDeferredContext() → Supplier 생성만 (아직 X)
   ↓
SecurityContextHolder.setDeferredContext()
   ↓
... (필터 체인 진행)
   ↓
어딘가에서 getContext() 호출  ❗
   ↓
Supplier.get() 실행됨 → 실제로 세션 접근해서 SecurityContext 로딩
```



#### 세션 저장소: HttpSessionSecurityContextRepository

**✔개념** 
- `SecurityContextRepository`의 기본 구현체
- SecurityContext를 HttpSession에 저장하고 꺼내는 역할
	- SecurityContext는 인증(Authentication)객체를 가지고 있다.
	- Spring Security의 기본 인증 시스템은 인증 객체를 session에 저장해두고 요청이 들어올 때마다 session에서 꺼내서 인증 상태를 유지하는 방식 

> [!INFO] session ➡ 서버 메모리에 저장되는 HttpSession 객체 
> - 사용자는 SeesionId를 가지고 있는거지 Session을 가지고 있는 것이 아니다.
> - 서버는 세션 ID를 확인하고 메모리에서 맞는 Seesion을 찾아낸 뒤 해당 세션의 SecurityContext를 확인
> - HttpSession : 서버가 인증 상태를 기억하는 방식 

#인증상태저장소

**✔주요 메서드**

**1‍⃣**loadDeferredContext()
```java
default DeferredSecurityContext loadDeferredContext(HttpServletRequest request) {  
    Supplier<SecurityContext> supplier = () -> 
		    this.loadContext(new HttpRequestResponseHolder(request, (HttpServletResponse)null));  
    return new SupplierDeferredSecurityContext(SingletonSupplier.of(supplier), SecurityContextHolder.getContextHolderStrategy());  
}
```
- `Supplier<SecurityContext>` 
	- SecurityContext를 **지연**해서 가져올 수 있는 객체
	- 이 코드에 세션에서 Context를 꺼내오는 로직이 들어있다.
	- 즉, 필요할 때까지 SecurityContext 생성을 미룰 수 있다.
- 이러한 지연 로딩은, 비동기 처리나 reacitve 환경같은 고성능 요청 처리에 적합한 메서드이다.
- 또한, 이전 버전과 달리 ServletResponse에는 의존하지 않아 유연성이 높아졌다. 

#성능최적화 #비동기처리 #reativeAPI


> [!WARNING] loadContext() -  loadDeferredContext의 이전 버전 
> - 6.x 이전에  deprecated 버전은 loadContext() 메서드였다. 
> - 💢 단점 
> 	- 하지만 과거 버전은 요청이 들어올 떄마다 즉시 SecurityCotext를 불러와야 했다.
> 	- 이는 비동기 환경과 맞지 않고 실제 필요할 때만 context를 부르는 lazy 로딩을 불가능케했다.
> 	- 유연성 부족 : `ServletRequest`와 `ServletResponse` 모두 읽어서 처리하는 점도 유연성이 떨어졌음




**2‍⃣** saveContext (context, request, response)
```java

```


### USER, PWD 재설정 

> 참고 : https://docs.spring.io/spring-boot/appendix/application-properties/index.html#appendix.application-properties.security
> ![[Pasted image 20250518132756.png]]  




매번 Password를 콘솔에서 확인하는 것은 고역이다.
따라서 application.properties를 통해 설정이 가능 
```yaml
spring:  
  security:  
    user:  
      name: admin   
      password: 1234
```


