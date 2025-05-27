![[Pasted image 20250520093706.png]]


> [!EXAMPLE] `ProviderManager` :  A Representative implementation  
> - Interface : AuthenticationManager 
> - implementatioin 
> 	- **ProviderManager** ⭐most commo
> 	- NoOpAuthenticationManager << for test
> 	- ObservationAuthenticationManager
> 	- etc 

## ProviderManger - implementation
```java
public class ProviderManager implements AuthenticationManager ....{
}
```

### Concept 
- The most commonly used **implementation**
- **Delegates** to a `List` of `AuthenticationProvider` instances.
- 만약 AuthenticationPriver List **모두 인증을 실패한다면, 예외**를 던질 것이다.
	- If none of the configured `AuthenticationProvider` instances can authenticate, authentication fails with a `ProviderNotFoundException`
	- Note : 각 AuthenticaionProvider는 ProviderManager가 던져준 Autentication의 type을 알고 그에 맞는 validation을 실행한다
![[Pasted image 20250525163346.png]]



method 
```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {  
    Class<? extends Authentication> toTest = authentication.getClass();  
    AuthenticationException lastException = null;  
    AuthenticationException parentException = null;  
    Authentication result = null;  
    Authentication parentResult = null;  
    int currentPosition = 0;  
    int size = this.providers.size();  
  
    for(AuthenticationProvider provider : this.getProviders()) {  
        if (provider.supports(toTest)) {  
            if (logger.isTraceEnabled()) {  
                Log var10000 = logger;  
                String var10002 = provider.getClass().getSimpleName();  
                ++currentPosition;  
                var10000.trace(LogMessage.format("Authenticating request with %s (%d/%d)", var10002, currentPosition, size));  
            }  
  
            try {  
                result = provider.authenticate(authentication);  
                if (result != null) {  
                    this.copyDetails(authentication, result);  
                    break;  
                }  
            } catch (InternalAuthenticationServiceException | AccountStatusException ex) {  
                this.prepareException(ex, authentication);  
                throw ex;  
            } catch (AuthenticationException ex) {  
                lastException = ex;  
            }  
        }  
    }  
  
    if (result == null && this.parent != null) {  
        try {  
            parentResult = this.parent.authenticate(authentication);  
            result = parentResult;  
        } catch (ProviderNotFoundException var12) {  
        } catch (AuthenticationException ex) {  
            parentException = ex;  
            lastException = ex;  
        }  
    }  
  
    if (result != null) {  
        if (this.eraseCredentialsAfterAuthentication && result instanceof CredentialsContainer) {  
            ((CredentialsContainer)result).eraseCredentials();  
        }  
  
        if (parentResult == null) {  
            this.eventPublisher.publishAuthenticationSuccess(result);  
        }  
  
        return result;  
    } else {  
        if (lastException == null) {  
            lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound", new Object[]{toTest.getName()}, "No AuthenticationProvider found for {0}"));  
        }  
  
        if (parentException == null) {  
            this.prepareException(lastException, authentication);  
        }  
  
        throw lastException;  
    }  
}
```
Purpose 
- iterate all the applicable authentication providers
- Cuz to **invoke theirs authenticate() method**
- **pass** "Authentication Object" 

Next : [[Flow 3. AuthenticationProvider and Manager, encoder]]