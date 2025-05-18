

> [!EXAMPLE] A typical implementation is ProviderManager
> - Interface ➡ AuthenticationManager 
> - implementatioin ➡ ProviderManager


```java
public class ProviderManager implements AuthenticationManager ....{
}
```
- AuthenticationManager's implementation


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


