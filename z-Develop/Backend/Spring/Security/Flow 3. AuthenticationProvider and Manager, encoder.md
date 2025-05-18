
default implementaion ➡ DaoAuthenticationProvider

> AuthenticationProvider has authenticate() method


✔**Role** (by take help otehr components)
- Delegate user info retrieval to `UserDetailsManager` or `UserDetailsService` **to load user details** from the storage system
- Once the user details are loaded, take help from the `password encoder` to **valdiate** **pwd**
- In short, it relies on other components to perform actual authentication logic.


### UserDetailsService/Manager
✔Role
- Responsible for loading user information based on the provided username.
```java
UserDetails loadUserByUsername(String username);
```
- Returns a `UserDetails` object containing the user's credentials and authorities.

✔UserDetailManager (extends UserDetailsService)
- additionally provides methods to create, update, delete, and check user existence.

>[!tip] UserDetailsService vs UserDetailManager
>- UserDetailsService : check user existence 
>- UserDetailManager : check user existence + create, update, delet, and 


