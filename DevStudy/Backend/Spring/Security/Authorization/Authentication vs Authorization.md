

### 차이 
✔**Authentication** (AuthN)
- 목적 : **유저의 identity를 check for 시스템 접근**
- 발생 : Authorization이 일어나기 전 
- 필요 데이터 : **login details** (generally)
- 에러 : 401 HTTP error
- 단순 시스템 접근을 위한 인증이기에 **유저에 따라 로그인 메커니즘을 바꿀 필요가 없다.**

✔**Authorization** (AuthZ)
- 목적 : **사람 or 유저의 권한을 check for 리소스 접근** 
- 발생 : Authentication이 끝난 뒤 
- 필요 데이터 : user's **privilege** or **roles** 
- 에러 : 403 HTTP error 





### How authorities stored in Spring Security 

#### 인터페이스 
GrantedAuthority라는 인터페이스가 있다. (메서드 1개)
```java 
/**Represents an authority granted to an Authentication object.
A GrantedAuthority must either represent itself as a String or be specifically supported by an AccessDecisionManager.
**/

public interface GrantedAuthority extends Serializable {

			String getAuthority(); // 단일 메서드 
```

#### 구현체 
There is 3 impl 
- JassGrantedAuthority
- **SimpleGrantedAuthority** ⭐ << most commonly used 
- SwitchUserGrantedAuthority

```java 
public final class SimpleGrantedAuthority implements GrantedAuthority {  
  
    private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;  
  
    private final String role;  
  
    public SimpleGrantedAuthority(String role) {  
       Assert.hasText(role, "A granted authority textual representation is required");  
       this.role = role;  
    }  
  
    @Override  
    public String getAuthority() {  
       return this.role;  
    }
```
- 만약 AuthZ를 store하고 싶다면 SimpleGrantedAuthority객체를 만들면 된다.(만들 때, 이름 넘겨주면 됨)


### 원리 이해 
#### ❓AuthZ가 언제 발생할까?
> UserDetails를 loading하면서 발생 


#### ❓Role or AuthZ의 이름은 언제 저장될까?
아래는 이전에 구현했던, userDetails를 loading하는 메서드이다.
```java
@Service  
@RequiredArgsConstructor  
public class BankUserDetailsService implements UserDetailsService {  
  
    private final CustomerRepository customerRepository;  
  
    @Override  
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {  
        Customer customer = customerRepository.findByEmail(username)  
                .orElseThrow(() -> new UsernameNotFoundException("User detials not found for the user" + username));  
        List<SimpleGrantedAuthority> authorities = List.of(new SimpleGrantedAuthority(customer.getRole()));  
        return new User(customer.getEmail(), customer.getPwd(), authorities);  
    }
```
- `return new User(.... authorities);`
	- 마지막 파라미터로 authrities를 받으며 UserDetails를 반환한다.
	- 여러 권한을 가질 수 있기 때문에 복수
	- 이렇게 넣은 authorities는 unmodifiedSet으로 저장된다.
	- UserDetails에는 authorities정보가 담겨 get메서드로 불러올 수 있다.
	```java 
	public interface UserDetails extends Serializable {  
			...
			Collection<? extends GrantedAuthority> getAuthorities();
	```

이렇게 반환한 UserDetails를 AuthNProvider가 받고 UsernamePasswordAuthenticationToken반환
- 여기에는 AuthZ포함되어있음 (unmodifiableList형태로 token에 저장됨)
- 반환한거는 Authentication인터페이스의 impl
```java
public class BankUsernamePwdAuthenticationProvider implements AuthenticationProvider {  
  
    private final BankUserDetailsService bankUserDetailsService;  
    @Override  
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {  
        ....
        UserDetails userDetails = bankUserDetailsService.loadUserByUsername(username);  
        return new UsernamePasswordAuthenticationToken(username, password, userDetails.getAuthorities());  
    }
```

- 조회 시 load메서드를 통해 customer의 role 필드를 참고해서 **authorities가 담긴 UserDetails를** UserDetailsService가 AuthenticationProvider에 전달해줌 


만약 Spring Security에서 **여러곳에서 AuthZ를 하려면 이렇게 반환한 token의 getAuthorities()메서드를 사용**하면 된다. ➡ 유연성

 


### Authoriy 객체 schema 
> Spring Security는 권한을 여러개 가질 수 있게 했다. 
> 개발자는 Authorization관련 테이블을 하나 더 만들어서 관리할 수 있다.

```sql 
[대충 authorities테이블 생성 스키마]

[대충 Insert 스키마]
```



src-sql에 추가 



### Authoriy 객체 만들기 
