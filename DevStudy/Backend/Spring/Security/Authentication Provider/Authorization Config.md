
### Authority를 위해 변경 

#### 1. Entity클래스 추가 및 수정 (Authority)

연관관계 주인 : Authority
 ```java
@Entity  
@Getter @Setter  
@Table(name="authorities")  
public class Authority {  
  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    private long id;  
  
    private String name;  
  
    @ManyToOne  
    @JoinColumn(name="customer_id")  
    private Customer customer;
```


연관관계 비주인 : Customer
```java 
@Entity  
@Getter @Setter  
public class Customer {  
	  ... 
  
    @OneToMany(mappedBy = "customer", fetch = FetchType.EAGER)  
    @JsonIgnore  
    private Set<Authority> authorities;
```


#### 2. UserDetailService 수정 
기존 
```java 
public class BankUserDetailsService implements UserDetailsService {  
  
    @Override  
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {  
        ....
        List<SimpleGrantedAuthority> authorities = List.of(new SimpleGrantedAuthority(customer.getRole()));  
        return new User(customer.getEmail(), customer.getPwd(), authorities);  
    }
```
- 기존에는 Authority를 배우기 전이라, role 필드를 활용했다.
- 이제 role 필드 안쓸거니 지우고 리팩토링하자 (db, 엔티티 클래스 전부 삭제)

>[!tip] 이제 Customer가 보유한 authorities를 활용 

변경 
- 메서드 변경 : getRole() ➡ getAuthorities()

```java 
		List<GrantedAuthority> authorities = customer.getAuthorities().stream()  
		    .map(authority -> new SimpleGrantedAuthority(authority.getName()))  
		    .collect(Collectors.toList());  
		return new User(customer.getEmail(), customer.getPwd(), authorities);
```


> [!WARNING] 위의 2개 했다고 Security의 Authorization이 자동으로 일어나지 않는다.


## Config AuthZ 

단순히 아래처럼 했다고 AuthZ 가 일어나지 않는다.
```java
    .addFilterAfter(new CsrfCookieFilter(), BasicAuthenticationFilter.class)  
    .requiresChannel(rcc -> rcc.anyRequest().requiresInsecure())  
    .authorizeHttpRequests((requests) -> requests  
            .requestMatchers("/myAccount", "/myCards", "/myLoans", "/myBalance", "/user").authenticated()  
            .requestMatchers("/notices", "/contact", "/register", "/error", "/invalid-session").permitAll());  
//http. formLogin(flc -> flc.disable());  
http.formLogin(withDefaults());
```

> 지금까지는 AuthN rule만 적용한거고, 이제는 AuthZ rule을 적용하자 

### 다양한 방법 

>[!EXAMPLE] 종류 
>1. 권한 단수 시 : .hasAuthority("권한명1")
>2. 권한 복수 시 : .has**Any**Authority("권한명1","권한명2")
>3. 복잡한 권한 설정 시 : access()  <<  SpEL 사용 

#### has~Authority 방법 
```java
.requiresChannel(rcc -> rcc.anyRequest().requiresInsecure())  
.authorizeHttpRequests((requests) -> requests  
    .requestMatchers("/myAccount").hasAuthority("VIEWACCOUNT")  
    .requestMatchers("/myCards").hasAuthority("VIEWCARDS")  
    .requestMatchers("/myLoans").hasAuthority("VIEWLOANS")  
    .requestMatchers("/myBalance").hasAuthority("VIEWBALANCE")  
    .requestMatchers("/user").authenticated()  
    .requestMatchers("/notices", "/contact", "/register", "/error", "/invalid-session").permitAll());

```
- 각 url별 AuthZ 설정 

>[!tip] .hasAuthority설정을 해 놓으면 **Authentication 과정은 ‘자동’으로 트리거**된다.
>복습
>1. `SecurityContextHolderFilter`가 먼저 실행
>	- 서블릿 호출 시 가장 먼저 `SecurityContextHolderFilter`가 먼저 실행된다.
>	- 이 filter가 서버에서 보유중인 httpSession(or 다른)저장소에서 SecurityContext를 꺼내와 `SecurityContextHolder`에 집어넣는다.
>	  
>2. AuthenticationFilter들이 등장
>	- UsernamePasswordAuthenticationFilter or BasicAuthenticationFiler에게 인증을 시도하게 넘긴다.
>	- 이제 security-flow 그림에서 봤던 인증 그림처럼 돌아갈 것이다.
>	  
>3. FilterSecurityInterceptor – **Authorization 검사**
>   
>   > 근데 여기서 **애초에 인증이 안 되어 있으면 2번 과정에서 Cut 당함** 


>[!tip] AuthorizationFilter가 실행되는 시점 
>- 인증을 끝낸 뒤 바로 실행
>	- 인증이 끝난 뒤 SecurityContextHolder에 성공 결과를 담은 AuthentcaionToken객체를 저장 
>	- 그 **다음 실행되는 것이 AuthorizationFilter**❗❗❗
>	- 이 필터는 AuthZ 규칙을 평가한다.

#### access() 방법 
다양한 방법이 있는데
1. path 에서 value를 extract하는 방법
2. multiple roles를 적용
	- access(allOf(ha~) 
	- new WebExpressionAuthorizationManager 사용 






- [x] 참고 : SET SQL 
```SQL
CREATE TABLE `authorities` (
  `id` int NOT NULL AUTO_INCREMENT,
  `customer_id` int NOT NULL,
  `name` varchar(50) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `customer_id` (`customer_id`),
  CONSTRAINT `authorities_ibfk_1` FOREIGN KEY (`customer_id`) REFERENCES `customer` (`customer_id`)
);
INSERT INTO `authorities` (`customer_id`, `name`)
 VALUES (1, 'VIEWACCOUNT');

INSERT INTO `authorities` (`customer_id`, `name`)
 VALUES (1, 'VIEWCARDS');

 INSERT INTO `authorities` (`customer_id`, `name`)
  VALUES (1, 'VIEWLOANS');

 INSERT INTO `authorities` (`customer_id`, `name`)
   VALUES (1, 'VIEWBALANCE');
```