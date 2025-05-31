
### Step

#### 1.  Config 수정 : API level 검증 삭제 
> Method-level로 검증할거이므로 컨트롤러의 검증 지우기 
> (AP)

requestMatcher에 적은 hasRole 삭제 
myLoans Path에 hasRole("USER") ➡ .authenticated()
```java 
		[기존]
		.requestMatchers("/myLoans").hasRole("USER")

		[변경]
		.requestMatchers("/myLoans").authenticated()
```

#### 2. Method 위에 @PreAuthorize 적용

서비스레이어에 적어도되는데, 지금은 간단한 예시라 controller-repository만 구현되어있다.
따라서 repo 메서드에 적용할 것 

```java
@Repository  
public interface LoanRepository extends JpaRepository<Loans, Long> {  
  
  @PreAuthorize("hasRole('USER')")  
  List<Loans> findByCustomerIdOrderByStartDtDesc(long customerId);
```


>[!tip] Prefix는 안 적어도 된다. 
>스프링 Security가 알아서 prefix를 append해준다.
>- ✅ `@PreAuthorize("hasRole('USER')")`
>- ❌ `@PreAuthorize("hasRole('ROLE_USER')")`



#### 3. 테스트 
[[5. Testing JWT token]] << 이거 참고해서 테스트해보기 
- 전체적인 흐름은 링크 페이지 참고
- 테스트 성공 실패 경우 나눠서 annotation 수정
	- 성공 : `@PreAuthorize("hasRole('USER')")` 
	- 실패 : `@PreAuthorize("hasRole('ROOT')")` 

실패하는 case의 경우 403 Error 발생 

Note : 2번 방법 말고도 Controller에 @PostAuthorize 적용해보기 
