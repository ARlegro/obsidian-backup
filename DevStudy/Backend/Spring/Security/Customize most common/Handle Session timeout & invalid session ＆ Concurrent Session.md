
> Once the login is completed, it is going to have default 30min session timeout 
> - That is, after 30min from login, need to re-login 

Also, Session timeout can be configured 

## How to configure session timeout 
![[supporter/image/PDF 12.png]]
### 1. setting yaml 
```yaml
server:  
  servlet:  
    session:  
      timeout: ${SESSION_TIMEOUT:10m}
```


Rule 
- If i **don't mention** any value (ex. 20), then it will be constrained **as Seconds**
- *High-security applications:* 2-5 minutes
- *Medium-security applications:* 15-30 minutes
- *Low-security applications:* Up to 1 hour

## Option : cofigure redirecting invalidsession 

>It's used in MVC
![[supporter/image/PDF 13.png]]

```java 
http.sessionManagement(smc -> smc.invalidSessionUrl("/invalidSession"))
```
When invalid session Id is submmitted(ex. session timeout), 
Redirects to the configured url path instead of login pag


## How to control the concurrent sessions 
>**By default**, spring security **is not going to enforce** any control on how many session and user can have 

In reality, 
- Sometimes, we want to restrict the *user to have only a single concurrent session*
- and other tiems want to restrict the *user to have 2~3 concurrent sessions*
- It depends on the situation 
>Let's try configure it 

### Configure maximum session
```java
http.sessionManagement( smc ->   
        smc.invalidSessionUrl("/invalid-session")  
            .maximumSessions(3)  
            
```

- If restrict the maximum session = 1, 
	- A user can maintain **only one active session at a time**
	- In other words, even if the user logs in successfully from 2 different browser, **only one session will remain valid** 
	- This is useful for **enhancing** **security**, especially in environments where account sharing is discouraged.


### Configure maxSessionPreventLogin

>[!tip] This setting determines *what happens when the max number of allowed session is reached*
![[supporter/image/PDF 14.png]]
>By default, this value is set to `false` 

```java
http.sessionManagement(smc -> smc.invalidSessionUrl("/invalidSession")
		.maximumSessions(1)
		.maxSessionsPreventsLogin(true))  // default = false
```

✅**If set to false(default)**
- The **previous session** will be **invalidated**, and new login will succeed.
- ❓When useful
	- the latest session should take precedence

✅**If set to true** 
- The **new login** attempt will be **blocked** 
- The user will remain logged in on the existing session
- ❓When useful
	- want to prevent simultaneous logins
	- ex. exam platform, banking apps 



### Configure Expired Url 
```java 

```
The URL to redirect to if a user tries to access a resource and session has been expired 

