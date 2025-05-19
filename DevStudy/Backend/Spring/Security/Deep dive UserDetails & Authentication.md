>[!QUESTION] Why exist two shcema, interfaces (Authentication && UserDetails)
> - Both of them, Representing user 
> - But Why do they exist ❓ 


![[PDF 6.png]]

2 Interfaces(Authentication && User) has similar kind of methods 
But some methods differ


**Authentication**
- This type of object has **methods related Authenticated** (unlike UserDetails)
- Representative implement : UsernamePasswordAuthenticationToken
- How can created? 
	- Made by AuthenticationProvider ex. Dao...
- Note : getDetails() return value is principal = Authenticated User Info ex. username, roles

**UserDetails** 
- has **some helpful methods** that Authentication does not 
	- Can use these methods regardeless of Wheater the authentication is successful or not

>[!EXAMPLE] Explain : Why seperate 2 Interfaces
>- **UserDetails's methods** that Authentication doesn't have is used only during actual authentication
>- **Authentication's methods** that UserDetails doesn' have is not used during actual authentication
>- **⭐By seperating them, can avoid carrying unnessary methods during process**




