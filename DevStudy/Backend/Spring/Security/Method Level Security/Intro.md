
지금까지 배운 spring의 filter chain은 주로 API paths/URLs를 대상으로 적용했다.
하지만 실제 비즈니스 로직에서는 같은 URL이라도 파라미터, 반환값, 객체 상태 등에 따라 접근 권한을 더 세밀하게 나눠야 할 수 있따.

> Method level에도 Spring Security를 적용할 수 있다.
  (특정 Layer도 가능)

### 필요한 이유 

#### 1. 세밀한 접근 제어 
실제 비즈니스 로직에서는 같은 URL이라도 파라미터, 반환값, 객체 상태 등에 따라 접근 권한을 더 세밀하게 나눠야 할 수 있다.

#### 2. 서비스/비즈니스 로직 보호 
> URL 매핑만으로는 애플리케이션 내부 호출을 (우회) 막지 못한다.

Service Layer에는 민감한 비즈니스 로직들이 들어있다.
근데 이 Layer는 웹 Layer뿐만 아니라 다른 경로로도 호출될 수 있다.
만약 서비스 계층에 보안을 적용하지 않으면, **다른 내부 호출이나 테스트 코드 등에서 우회 접근이 가능**

따라서, 메서드 레벨 보안은 실제 비즈니스 로직이 위치한 **서비스 계층에서 일관된 보안 정책을 강제**

>[!tip] Method level security는 non-web applications(엔드포인트 없는)에도 security 적용이 가능
>즉, http 없이도 동일한 검증을 항상 수행 

#### 복잡한 권한 조건 처리
- `@PreAuthorize` 등은 SpEL을 통해 파라미터, 반환값, 인증 객체 등 다양한 정보를 조합해 복잡한 권한 조건을 선언적으로 작성할 수 있다.

### 보안 방식 유형 
> 2가지 유형으로 나뉜다.

#### 1. Invoccation AuthZ (호출 권한 검증)
> 메서드를 **누가 호출할 수 있는지**를 검증함
- 특정 역할/권한을 가진 사용자만 호출 가능하게 제한 
- 예시
	```JAVA
	@PreAuthorize("hasRole('ADMIN')")
	public void deleteUser(Long id) { ... }
	```

#### 2. Filtering AuthZ (파라미터/리턴값 필터링 권한)
>Method 파라미터에 **어떤 데이터를 전달할 수 있는지**
>Method의 **Return 값** 중 어떤 데이터를 받을 수 있는지**를 검증


### 설정
![[supporter/image/PDF 23.png]]
```java 
public @interface EnableMethodSecurity {  
  boolean prePostEnabled() default true;  
  
  boolean securedEnabled() default false;  
  
  boolean jsr250Enabled() default false;  
  
  boolean proxyTargetClass() default false;  
  
  AdviceMode mode() default AdviceMode.PROXY;  
  
  int offset() default 0;
```

By default, `prePostEnabled()` 빼고는 전부 `false`

- `prePostEnabled`
	- true :  `@PreAuthorize`, `@PostAuthorize` 를 사용가능하다.
	- 이거 많이 쓰인다.
- `securedEnabled`
	- `true` : `@Secured` 사용 가능 
- `jsr250Enabled` 
	- `true` : `@RoleAllowed` 사용 가능 


>[!tip] @Secured, @RollAllowed 보다는 차라리 @PreAuthorize, @PostAuthorize를 많이 씀 
>- @Secured, @RollAllowed 는 레거시 





### 설정 
`@EnableMethodSecurity` on top of spring boot main class
(deprecated anno = `@EnableGlobalMethodSecurity`)

테스트에서는 @Secured, @RoleAllowed 전부 사용해볼거니 true 로 
```java 

```



