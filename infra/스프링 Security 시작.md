
로그인 창 뜸 


### 자동으로 세션을 기억

>[!QUESTION] 왜 한번 로그인 하면 그 이후부터는 물어보지 않게 됨?
>- 스프링 시큐리티는 내부적으로 로그인한 세션을 기억

#### 원리 
1. 로그인 성공 ➡ 인증 정보가 SecurityContextHolder에 저장
2. `SecurityContextPersistenceFilter`가 이를 **세션에 저장**
3. 이후 요청이 들어오면, 세션에 저장된 인증 정보를 다시 꺼내서 `SecurityContextHolder`에 넣음.
4. 따라서 사용자는 계속 로그인 상태를 유지함.

#### SecurityContextHolderFilter

> 참고 : deprecated 버전 : SecurityContextPersistenceFilter 

- 인증 상태 유지의 핵심 역할 


#### 세션 저장소: HttpSessionSecurityContextRepository

>[!EXAMPLE] 개념
>- `SecurityContextRepository`의 기본 구현체
>- 주요 메서드
>	- loadContext() : 
