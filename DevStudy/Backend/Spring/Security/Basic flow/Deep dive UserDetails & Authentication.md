>[!QUESTION] Why exist two shcema, interfaces (Authentication && UserDetails)
> - Both of them, Representing user 
> - But Why do they exist ❓ 

![[supporter/image/PDF 17.png]]

2 Interfaces(Authentication && User) has similar kind of methods 
But some methods differ

**Authentication**
- This type of object has **methods related Authenticated** (unlike UserDetails)
- Representative implement : `UsernamePasswordAuthenticationToken`
- **How can created?** 
	- **Made by** `Spring Security Filter` (authenticated = false) ex. 
	- **Updated** authenticated status **by AuthenticationProvider** 
- Note : getDetails() return value is principal = Authenticated User Info ex. username, roles

**UserDetails** 
- has **some helpful methods** that Authentication does not 
	- Can use these methods regardeless of Wheater the authentication is successful or not

>[!EXAMPLE] Explain : Why seperate 2 Interfaces
>- **UserDetails's methods** that Authentication doesn't have is **used only** during actual authentication
>- **Authentication's methods** that UserDetails doesn' have is **not used** during actual authentication
>- **⭐By seperating them**
>	1. SRP 
>		- UserDetails : only focus on user info 
>		- Authentication : only focus on authentication status && result 
>		- can avoid carrying unnessary methods during process
>	2. Flexiable

