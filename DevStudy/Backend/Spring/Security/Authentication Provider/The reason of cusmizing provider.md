
How to define our own Authentication Provider

> In builted Provider is `DaoAuthentication Provider`
> - This is good job 
> - But some sceniro require to custom Provider 

### Why do i need config provider 

When organization want to **provide various type of authentication**
ex. name&pw or OAUTH2 or JAAS
- If i want to authentication **based on age or country**, default Provider is not going to help me 
- UserDetailService that Provider rely on only proivdes loading-features (Not help to validate correct age or country)
- So, In this case, we need to cusmize own Authenticatio Provider
	- Once UserDetails are loaded by UserDetails service, **Customized Provider will additional check** of Authentication

### Understanding methods 
❗There are two Authentication Provder (MYSQL && Spring Sercurty). Choose Spring Security version 

#### Interface 
```java 
public interface AuthenticationProvider {  
  
		Authentication authenticate(Authentication authentication) throws AuthenticationException;  
  
    boolean supports(Class<?> authentication);
```
✅**Only 2 methods**
1. **authenticate(...)**
	- Perform **actual authentication logic**
	- ProviderManager invoke this method
	- For authenticate(valiate), provider need to loading userdetails
	- Return : add info indicating whether authentication is successful or not 
	  
2. **supports(...)**
	- **Indicating whether this style of authentication is supported**
	- ex. Dao~Provider support methods() is bellow: `UsernamePasswordAuthenticationToken`
	```java
	public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
	...
	
	public abstract class AbstractUserDetailsAuthenticationProvider  ...{

		@Override  
		public boolean supports(Class<?> authentication) {  
		    return (UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));  
		}
	```
	- If ProviderManager receive UsernamePasswordAuthenticationToken, then ProviderManager will invoke DaoAuthenticationProvider
		![[Pasted image 20250521132202.png]]
> DaoAutheticationProvider is provider which support usernamePassword style 


### TestingAuthenticationProvider for Testing 
>[!tip] Class `TestingAuthenticationProvider`, `TestingAuthenticationToken`  is for unit Testing
```java
>@Override  
public Authentication authenticate(Authentication authentication) throws AuthenticationException {   
```
-  Nothing to happen, Input = output


### Customize
```java 
@Component  
@RequiredArgsConstructor  
public class BankUsernamePwdAuthenticationProvider implements AuthenticationProvider {  
  
    private final BankUserDetailsService bankUserDetailsService;  
    private final PasswordEncoder passwordEncoder;  
    @Override  
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {  
        String username = authentication.getName();  
        String password = authentication.getCredentials().toString();  

				// userDetailsService
        UserDetails userDetails = bankUserDetailsService.loadUserByUsername(username);  
		    
		    // PasswordEndoer : 평문으로 들어오느 로그인 pwd를 hash된 db pwd와 비교
        boolean isMatch = passwordEncoder.matches(password, userDetails.getPassword());  
        if (!isMatch){  
            throw new BadCredentialsException("Bad credentials");  
        }  
  
        return new UsernamePasswordAuthenticationToken(username, password, userDetails.getAuthorities());  
    }  
  
    @Override  
    public boolean supports(Class<?> authentication) {  
        return (UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));  
    }  
    //`AuthenticationProvider` 가 처리할 토큰 타입을 지정
}
```


✅Step1. need to load UserDetails 
- 방법1. Invoke UserDetailsService's method
- 방법2. Just can use UserDetailsService's load() method as it 
- pwd : authetication.getCredentials().toString()

✅Step2. matching pwd By passwordEncoder
- If fail ➡ throw BadCredentialsException
- **If success**  
	- Valdiate 
	- return UsernamePasswordToken(username, pwd, authorities)

>[!tip] public boolean supports(Class ~)  의미 
>AuthenticationProvider가 처리할 토큰 타입을 지정

