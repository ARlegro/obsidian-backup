
> CSRF (Cross-Site Request Forgery)

This is security **attack** unlike CORS that leveraged by the **hackers to steal the data**

## What is CSRF attack 
`CSRF = Cross-Site Request Forgery`
*forgery = 위조*

#### Concept 
- A CSRF attack **tricks** a logged-in user into **unknowingly submitting** a request to a web application where they are **already authenticated**.
- Attacker do **request using the victim’s credentials** (like session cookies), **without user's identity**(ex. username, pw)

> 다른 사이트의 요청을 속여서 보내는 공격
> 다른 사용자의 인증된 세션을 활용해 서버에 요청을 들어오게 만드는 것 

>[!EXAMPLE] 개념 정리 
>- CORS와 달리, 해커가 데이터를 훔치거나 사용자의 의도와는 다른 행동을 유발하기 위해 악용하는 보안 공격 유형 
>- 공격자는 사용자의 인증 정보를 활용하여 사용자의 Identity를 탈취하지 않고도 사용자인 것처럼 위장(forgery)해서 요청을 보낸다.




#### Scenario 
1. 사용자 로그인
	- The user is **already logged in** to a specific site (ex. bank, samsung.com)
2. 악성 웹사이트 방문 유도
	- The attacker tricks the user into visiting a **malicious website** 
3. 악성 요청 전송 : That malicious site silently sends a **request to the legitimate server** using the victim’s **authenticated session** (via cookies).
4. 인증된 세션 활용 
	- Since the browser **automatically includes cookies**, the server thinks the request is legit.
	- note : Cookie contains domain name 
- **The action is executed without the user’s awareness.** 

> [!WARNING] No Need to steal credentials 
> - CSRF doesn't directly steal teh user's identity(ex. username, pw)
> - But it exploits the user to carry put an action 

![[supporter/image/PDF 15.png]]

```html 
 [잘못된 요청을 일으키는 링크]
  <form action="https://samsung.com/changePassword" method="POST" id="form">
    <input type="hidden" name="password" value="hakcer@evil.com">
  </form>

	<script>
    document.getElementById("form").submit()
  </script>
```
- If the user is already logged into `samsung.com`,  
-  and visits `evil.com` containing the above code,  
- the browser sends the session cookie (e.g., `JSESSIONID`) with the request.  
- The server executes the request, thinking it came from the legitimate user.

백엔드 서버 입장에서는 올바른 쿠키 세션이기 때문에 정상적인 요청이라고 생각하고 작동한다.
이렇게 embedded된 form 형식은 사용자가 쉽게 알아차릴 수 없어서 해커가 해킹할 시간을 충분히 준다

#### Conclusion
- CSRF **doesn’t steal** your cookies.  
- It **abuses** the fact that your **browser automatically sends cookies**,


---
## Default fonfig to protect CSRF in Spring Security

### Default 

> By Default, Spring Security is going **to secure** from requests(post, update, delete etc that alter data)
> for protecting CSRF attack

기본적으로 Spring Security는 데이터를 변경하는 요청으로부터 CSRF 공격을 방어하도록 설정 
❓How❓
- 서버가 요청을 받을 때, 세션 쿠키 외에 **추가적인 CSRF 토큰이 유효한지 확인**하는 방식으로 작동


>[!WARNING] Disabling the CSRF protection **is not recommended** 
>Before learing CSRF, I did configure csrf.disable()
```java 
http.
	...
	.csrf(csrfConfig -> csrfConfig.disable())
```

**Need to properly implementation a solution** to handle CSRF attack 

### ❓What happens when remove custom csrf config  and post 
```java 
[Remove]
.csrf(csrfConfig -> csrfConfig.disable())
```

✅Result  #Invalide-CSRF-Token
```json
{

    "timestamp": "2025-05-23T17:39:09.598952100",
    "status": 403,
    "error": "Forbidden",

    "message": "Invalid CSRF Token 'null' was found on the request parameter '_csrf' or header 'X-CSRF-TOKEN'.",

    "path": "/register"

}
```
Spring security 
- is trying to invalidate session so that i can't create data 
- securing all posts, put, delete etc by throwing a 403 error
- Note : GET method is allowed by default in Spring Security

> Whenever the CSRF token is missing, 
> The Spring Security will assume that someone is trying to attack application 


[[CSRF Solution]]
---

