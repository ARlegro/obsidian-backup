![[PDF 4.png]]

### AuthenticationProvider
>`DaoAuthenticationProvider` : default implementaion
>AuthenticationProvider has authenticate() method


✔**Role** (by take help otehr components)
- **Delegate user info** retrieval **to** `UserDetailsManager` or `UserDetailsService` **to load user details** from the storage system
- Once the user details are loaded, take help from the `password encoder` to **valdiate** **pwd**
- In short, it relies on other components to perform actual authentication logic.


### UserDetailsService/Manager
![[PDF 2.png]]

**✔UserDetailService** - Interface
- Responsible for **loading user information** based on the provided username.
- **Read-Only** Responsibility 
	- Authentication process to look up user details 
	- ❌ CURD ❌
	- only **solo method**
- Returns a `UserDetails` object containing the user's credentials and authorities.
```java
UserDetails loadUserByUsername(String username);
```

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


![[PDF.pdf#page=18&rect=23,4,1323,697|PDF, p.18]]





