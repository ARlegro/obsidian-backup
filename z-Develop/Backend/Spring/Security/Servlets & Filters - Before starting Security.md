

## Before start Security : Learn Servlets & Filters
![[PDF.pdf#page=12&rect=43,213,742,586|Spring_Security, p.12]]

>[!tip] Why try to learn **Sevlet, Filter** When Learning Spring Security❓
>- A lot fo function in Spring Security is implemented With the **help of filters and Servlets**

### Servlet 

#### why servlet ❓ (based client ↔ server flow)
- When Client is trying to send a request with the help of browser, ther request is going to come to server in protocla(ex. HTTP, HTTPS)
- But without servlet, the server can't make sense that server have returned 

#### Sevlet Container ⭐ ex. Tomcat, JBoss 

java is object programming ➡ anything we want has to be **converted** into **java object**

**Role** 
- From the **request** perspective 
	1. **Convert** : HTTP messages ➡ SevletRequests  
	2. **Hand over** : hand over ServletRequests to Servlet method as a parameter

- From the **response** perpective (Similar)
	1. **Recieve** 
		- Sevlet make output to ServletResponse object. and hand over it to Servlet Container
		- that is, Servlet container receive ServletReponse from Servlet 
	2. **Convert** : ServletRequests ➡ HTTP(s) messages 

> [!INFO]
> - ServletReuqests is object, and has data sent by client application
> - Servlet is leveraged by the framework(ex. spring boot, spring security)
> - The frameworkd make **aggresive use of servlets**  

>[!SUCCESS]  Thanks to Framework, the developer only focus on Business logic
>- no worry about how to build a sevlet, how to build a fileter
>- Everything is begin taken care by the framework


### Filter
>[!tip]  Before request reaching to the sevlet, there are filteres in between 

- filter is inside java web applications
- **Intercept every request and response**
	- identify if the user is authenticated or not
	- if not, redirect user to login page
- **main implementaion in securty related logic** 
- Enfocing security

> [!Warning] Filter only used spring security

>[!tip] There are a lot of filter chain in Interal Architecture of Spring Security 
>because a lot of purpose 
>- Authentication
>- Authorization
>- Session managing
>- protect CSRF
>- CORS
>- logout
>- etc




## Internal Flow 
> first of all, focus on Overally flow (the details later)

![[PDF.pdf#page=13&rect=57,105,1398,647|Spring_Security, p.13]]





