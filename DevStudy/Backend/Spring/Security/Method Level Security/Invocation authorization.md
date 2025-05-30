
> [!info] Invocation AuthZ은 2가지 중요 annotation을 가지고 있다.
> 1. @PreAuthorized
> 2. @PostAuthorized



### @PreAuthorized 

자바 메서드 위 

![[supporter/image/PDF 24.png]]
이렇게 명시해두면 이 메서드가 실행되기 전에 Spring Security에서는 validation을 진행한다.
validation이 실패하면 이 메서드는 실행되지 않는다.

`#username == authentication.principal.username`
- 오직 로그인 된 유저만 이 메서드를 실행할 수 있게 설정 

### @PostAuthorize 
메서드가 실행되는동안 어떤 Authorization도 일어나지 않는다.
메서드가 전부 실행되고 나서 Authorization Check가 일어난다.
![[supporter/image/PDF 25.png]]

**❓언제 쓰일까??**
- **반환값에 따라 접근제어가 필요한 경우** 
	- ex. 게시글 상세 조회 시 ➡ 반환된 게시글이 현재 로그인한 사용자와 일치할 때만 접근 허용 
	```JAVA
	@PostAuthorize("returnObject.username == authentication.pincipal.name") 
	public Post getPost(Long id) { return postRepository.findById(id);
	```
- **중간 로직 결과에 따라** 권한을 결정해야 할 때 
	- ex. 외부 API 호출 결과, DB 조회 데이터 결과 등 
- **주로 조회에 적합** 
	- 쓰기 작업은 권한을 미리 확인하는게 좋음 



### 원리 
>Spring은 AOP를 이용해서 이러한 annotaion들을 작동시킴 


#### @PreAuthorize 관련 Interceptor

```java 
public final class AuthorizationManagerBeforeMethodInterceptor implements AuthorizationAdvisor {


		public static AuthorizationManagerBeforeMethodInterceptor preAuthorize() {  
		  return preAuthorize(new PreAuthorizeAuthorizationManager());  
		}  
		  
		public static AuthorizationManagerBeforeMethodInterceptor preAuthorize(PreAuthorizeAuthorizationManager authorizationManager) {  
		  AuthorizationManagerBeforeMethodInterceptor interceptor = new AuthorizationManagerBeforeMethodInterceptor(AuthorizationMethodPointcuts.forAnnotations(new Class[]{PreAuthorize.class}), authorizationManager);  
		  interceptor.setOrder(AuthorizationInterceptorsOrder.PRE_AUTHORIZE.getOrder());  
		  return interceptor;  
		}  
		  
		public static AuthorizationManagerBeforeMethodInterceptor preAuthorize(AuthorizationManager<MethodInvocation> authorizationManager) {  
		  AuthorizationManagerBeforeMethodInterceptor interceptor = new AuthorizationManagerBeforeMethodInterceptor(AuthorizationMethodPointcuts.forAnnotations(new Class[]{PreAuthorize.class}), authorizationManager);  
		  interceptor.setOrder(AuthorizationInterceptorsOrder.PRE_AUTHORIZE.getOrder());  
		  return interceptor;  
		}
```
preAuthorize 메서드를 활용해서 method-level security를 적용시킴 

#### @PostAuthorize 관련 Interceptor
AuthorizationManagerAfterMethodInterceptor

```java
public final class AuthorizationManagerAfterMethodInterceptor implements AuthorizationAdvisor {
			....
		public static AuthorizationManagerAfterMethodInterceptor postAuthorize() {  
  return postAuthorize(new PostAuthorizeAuthorizationManager());  
}  
  
		public static AuthorizationManagerAfterMethodInterceptor postAuthorize(PostAuthorizeAuthorizationManager authorizationManager) {  
		  AuthorizationManagerAfterMethodInterceptor interceptor = new AuthorizationManagerAfterMethodInterceptor(AuthorizationMethodPointcuts.forAnnotations(new Class[]{PostAuthorize.class}), authorizationManager);  
		  interceptor.setOrder(AuthorizationInterceptorsOrder.POST_AUTHORIZE.getOrder());  
		  return interceptor;  
		}  
		  
		public static AuthorizationManagerAfterMethodInterceptor postAuthorize(AuthorizationManager<MethodInvocationResult> authorizationManager) {  
		  AuthorizationManagerAfterMethodInterceptor interceptor = new AuthorizationManagerAfterMethodInterceptor(AuthorizationMethodPointcuts.forAnnotations(new Class[]{PostAuthorize.class}), authorizationManager);  
		  interceptor.setOrder(AuthorizationInterceptorsOrder.POST_AUTHORIZE.getOrder());  
		  return interceptor;  
		}

```

이 클래스 @PostAuthorize 관련된 로직들을 전부 갖고 있다.



