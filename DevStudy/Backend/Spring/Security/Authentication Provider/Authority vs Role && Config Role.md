

### Authority vs Role

### 비교 

|       | Authority                           | Role                              |
| ----- | ----------------------------------- | --------------------------------- |
| 정의    | **individual** privilege or action  | a **group** of privileges/actions |
| 제어 방식 | 세분화된 접근 제어                          | 대략적인 접근 제어                        |
| 예시    | `READ_PRIVILEGE`, `WRITE_PRIVILEGE` | `ROLE_ADMIN`, `ROLE_USER`         |
|       |                                     |                                   |

만약 Authoritiy가 100개가 넘는다면 관리하기 힘들 것이다.
따라서 Role도 필요 


>[!tip] Authority 랑 Role 헷갈리지 않게 Role 부여 시 prefix로 커스텀 가능 ex. ROLE_ADMIN

#### prefix config하는 법 : GrantedAuthorityDefaults 빈 재정의 
> 아래 클래스를 응용하면 된다.
```java
public final class GrantedAuthorityDefaults {  
  private final String rolePrefix;  
  
  public GrantedAuthorityDefaults(String rolePrefix) {  
    this.rolePrefix = rolePrefix;  
  }  
  
  public String getRolePrefix() {  
    return this.rolePrefix;  
  }  
}
```


```java 
@Bean  
static GrantedAuthorityDefaults grantedAuthorityDefaults() {  
  return new GrantedAuthorityDefaults("ROLE_");  
}
```

> [!WARNING] @Bean static GrantedAuthorityDefaults …로 선언해야한다 ⭐
> 
> Need to expose GrantedAuthorityDefaults using a static methods
> ❓Why?
> - `GrantedAuthorityDefaults`는 **컨테이너 가장 이른 시점**에 필요하다.
> - Spring Security는 Role-prefix를 확인하려고 `GrantedAuthorityDefaults` 빈을 즉시 찾는데, 빈 정의가 아직 없으면 기본값을 그대로 사용한다.
> - But static이면?
> 	- static @Bean 메서드는 @Configuration 객체를 생성하지 않고도 바로 실행할 수 있다.
> 	- `GrantedAuthorityDefaults` 정의·생성이 **컨테이너 부팅 가장 앞단**에서 이뤄진다.
> 	  
> > 덕분에, Spring Security 내부구성이 초기화될 때 원하는 prefix를 즉시 참조할 수 있다.

## Role Config 
### Step1. Role 추가 
일단 이전에 설정해 놓은 authorities테이블의 데이터들은 전부 삭제 후 추가 
```sql
DELETE FROM `authorities`;

INSERT INTO `authorities` (`customer_id`, `name`)
VALUES (1, 'ROLE_USER');

INSERT INTO `authorities` (`customer_id`, `name`)
VALUES (1, 'ROLE_ADMIN');
```

### Step2. hasRole() or hasAnyRole() 사용 


```java
.authorizeHttpRequests((requests) -> requests  
    .requestMatchers("/myAccount").hasRole("USER")  
    .requestMatchers("/myCards").hasAnyRole("ADMIN", "USER")  
    .requestMatchers("/myLoans").hasRole("USER")  
    .requestMatchers("/myBalance").hasRole("USER")  
    .requestMatchers("/user").authenticated()
```


>[!tip] security에서 role을 config할 때 prefix를 안 적어도 자동으로 적용해준다.
>- Cuz hasRole()메서드를 invoke할 때마다 내부적으로 prefix를 append하는 로직이 있다.
>- 만약 prefix를 적으면 prefix가 2번 적용되어 403 error가 발생할 것이다.
>- prefix 명시가 필요한 경우는 DB에 ROLE을 저장할 때 뿐 




### EventLister 활용 : 403 에러 
> 어떤 메시지를 던질지, redirect를 할지 등 403 error handle 로직을 개발자가 정할 수 있따.
https://docs.spring.io/spring-security/reference/servlet/authorization/events.html

실패 시 : AuthorizationDeniedEvent 이용 
```java 
@Component  
@Slf4j  
public class AuthorizationEvents {  
  
  @EventListener  
  public void onFailure(AuthorizationDeniedEvent event) {  
    log.error("Authorization failed for the user : {} due to : {}", event.getAuthentication().get().getName(),  
        event.getAuthorizationResult().toString());  
  }  
```

**❓왜 성공시에는 EvenetLister를 사용하지 않는가?**
- it'll be too noisy inside app to listen all to those events
- 성공이면 성공이지 무슨 성공시마다 event발생하게하면 성능 문제 발생
- 에러는 가끔이고 일반적인 로직은 하루에 몇천번 몇만번 일어나는데 애바지 
- 따라서 성공 lister는 비추천 

> 굳이 하고 싶다면 https://docs.spring.io/spring-security/reference/servlet/authorization/events.html들어가서 직접 구현 ㄱ 




