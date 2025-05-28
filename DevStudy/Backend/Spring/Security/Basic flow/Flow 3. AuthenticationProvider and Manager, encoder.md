![[PDF 4 1.png]]

### AuthenticationProvider
>`DaoAuthenticationProvider` : default implementation
>AuthenticationProvider has authenticate() method

✔**Role** (by take help otehr components)
- **Delegate user info** retrieval **to** `UserDetailsManager` or `UserDetailsService` **to load user details** from the storage system
- Once the user details are loaded, take help from the `password encoder` to **valdiate** **pwd**
- In short, it relies on other components to perform actual authentication logic.

#### Impl - DaoAuthenticationProvider
>Use a `UserDetailsService` and `PasswordEncoder` to authenticate a username and password.

```java 
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
	
```

![[Pasted image 20250525170759.png]]
- Looks up the `UserDetails` from the `UserDetailsService`.
- Uses the `PasswordEncoder` to validate the password on the `UserDetails` returned in UserDetailsService.
- If success
```java 
@Override  
protected Authentication createSuccessAuthentication(Object principal, Authentication authentication,  
       UserDetails user) {  
    String presentedPassword = authentication.getCredentials().toString();  
    boolean isPasswordCompromised = this.compromisedPasswordChecker != null  
          && this.compromisedPasswordChecker.check(presentedPassword).isCompromised();  
          
    if (isPasswordCompromised) {  
       throw new CompromisedPasswordException("The provided password is compromised, please change your password");  
    }  
    
    boolean upgradeEncoding = this.userDetailsPasswordService != null  
          && this.passwordEncoder.upgradeEncoding(user.getPassword());  
    if (upgradeEncoding) {  
       String newPassword = this.passwordEncoder.encode(presentedPassword);  
       user = this.userDetailsPasswordService.updatePassword(user, newPassword);  
    }  
    // super.createSuccessAuthentication(...)` 호출
    return super.createSuccessAuthentication(principal, authentication, user);  
}

//[SUPER의 createSuccessAuthentication]
protected Authentication createSuccessAuthentication(Object principal, Authentication authentication, UserDetails user) {
    // credentials는 null 처리
    UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
            principal, null, user.getAuthorities());
    result.setDetails(authentication.getDetails());
    return result;
}

```
- principal = userDetails 객체
- null 넣는 이유 : 비밀번호 제거 (인증했으니)
- result.setDetails(authentication.getDetails() : 인증 객체에 인증 성공 정보 넣음 
- 이렇게 반환한 Token은 다시 `AuthenticationFilter`가 SecurityContextHolder에 저장한다

### UserDetailsService/Manager
![[PDF 2 1.png]]

**✔UserDetailService** - Interface
>Invoked by provider
```java 
public interface UserDetailsService {

		UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```


- Responsible for **loading user information** based on the provided username.
	- retrieving a username, pw, and other attributes for authenticating 

- Only have **one** **Read-Only** method   
	- Authentication process to look up user details 
	- ❌ CURD ❌
	  
- The type of Impl 
	1. In-memory
	2. JDBC 
	3. caching 
	   
- Return : `UserDetails` object containing the user's credentials and authorities.

**✔UserDetailManager (extends UserDetailsService)** - Interface
- **additionally provides methods** to create, update, delete, and check user existence.
- Implementation
	1. `InmemoryUserDetailManager` : In-Memory
	2. `JdbcUserDetailsManager` : DB
	3. `LdapUserDetailsManager` : leagcy 
>[!tip] UserDetailsService vs UserDetailManager
>- UserDetailsService : check user existence 
>- UserDetailManager : check user existence + create, update, delet, and 



>[!QUESTION] Why seperate UserDetailsService vs UserDetailsManager
> #seperate #UserDetailsServie  #UserDetailsManager  #srp
>- Spring Security follows the principle of Seperation of Concerns
>1. **SRP** 
>	- UserDetailsService : only loads user data (**Authentication**)
>	- UserDetailsManager : handle user lifecycle (**CURD**)
>2. **Securiy** & **Maintainability** 
>	- In many system, user data is read-only
>	- In this case, UserDetailsManager is unnecessary
>3. **Flexibility** 



### If successcful 
1. **Successful authentication details will be stored in SecurityContext**
2. After that, invoke the REST API 
3. REST API's respose will go to filters 

![[supporter/image/PDF 18.png]]




