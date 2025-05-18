
로그인 창 뜸 


### 자동으로 세션을 기억

>[!QUESTION] 왜 한번 로그인 하면 그 이후부터는 물어보지 않게 됨?
>- 스프링 시큐리티는 내부적으로 로그인한 세션을 기억

#### 원리 
```java
package sun.security.util;  
  
import java.security.AccessController;  
import java.security.PrivilegedAction;  
import java.security.Security;  
  
public class SecurityProperties {  
  
			public static final boolean INCLUDE_JAR_NAME_IN_EXCEPTIONS  
			    = includedInExceptions("jar");  
			  
			/**  
			 * Returns the value of the security property propName, which can be overridden * by a system property of the same name * * @param  propName the name of the system or security property  
			 * @return the value of the system or security property */
			public static String privilegedGetOverridable(String propName) {  
			    if (System.getSecurityManager() == null) {  
			        return getOverridableProperty(propName);  
			    } else {  
			        return AccessController.doPrivileged((PrivilegedAction<String>) () -> getOverridableProperty(propName));  
			    }  
			}  
			  
			private static String getOverridableProperty(String propName) {  
			    String val = System.getProperty(propName);  
			    if (val == null) {  
			        return Security.getProperty(propName);  
			    } else {  
			        return val;  
			    }  
			}  
			  
			/**  
			 * Returns true in case the system or security property "jdk.includeInExceptions" * contains the category refName * * @param refName the category to check  
			 * @return true in case the system or security property "jdk.includeInExceptions" *         contains refName, false otherwise */
			 * public static boolean includedInExceptions(String refName) {  
			    String val = privilegedGetOverridable("jdk.includeInExceptions");  
			    if (val == null) {  
			        return false;  
			    }  
			  
			    String[] tokens = val.split(",");  
			    for (String token : tokens) {  
			        token = token.trim();  
			        if (token.equalsIgnoreCase(refName)) {  
			            return true;  
			        }  
			    }  
			    return false;  
			}
```
- 라이브러리가 제공해주는 SecurityProperties를 보면 내부적으로 코드검증이 일어난다.






