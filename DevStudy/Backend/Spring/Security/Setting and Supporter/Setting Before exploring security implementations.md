
### Logging 

logging only for security 
```yaml
logging:  
  ...
  level:  
    org.springframework.security: ${SPRING_SECURITY_LOG_LEVEL:TRACE}
```


### Check

✔ Check "Build Tools ➡ Maven ➡Importing" : source, doc, annotation

![[Pasted image 20250518202911.png]]
By this, Whenever add a new dependency, Maven is going to automatically take care of downlading source files 

###  Warp New terminal 

is getting popular these days 
Cuz 
1. UI 
2. AI 


### SQLECTRON
- super lightweight client


### Opensssl 



### 프론트 서버 실행 단축키 : npx
ng serve ➡ npx ng serve
- npx가 로컬 cli를 알아서 찾아준다.
- 글로벌 경로(윈도우)가 아닌 로컬 프로젝트(ubuntu)안에 설치된 Angular CLI의 ng를 우선 사용
- WSL 환경에서는 window용 node.exe를 직접 실행 못하니 오류된 것