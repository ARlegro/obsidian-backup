
CUSTOMER(/api/customer)
- 로그인 
- 로그아웃 

NOTICE 
- 전체 공지 
- 관리자가 쓴 글 모두가 보도록 

NEWS 
- 

DASHBOARD 🔐
- DF



### Customer (/api/customer)

| 메서드  | URL         | BODY                      | 내용      | Security 허용 범위 |
| ---- | ----------- | ------------------------- | ------- | -------------- |
| POST | /           | usernmae, email, password | 유저 등록   | permitAll()    |
| POST | /login      | email, password           | 헤더로 JWT | permitAll()    |
| POST | /json-login | email, password           | 바디로 JWT | permitAll()    |
|      |             |                           |         |                |
|      |             |                           |         |                |

### Notice (/api/notice)
공지 

| 메서드    | URL | BODY | 내용     | Security 허용 범위 |
| ------ | --- | ---- | ------ | -------------- |
| POST   |     |      | 공지 등록  | admin계정만       |
| GET    |     |      | 공지 최근꺼 | permitAll()    |
| PATCH  |     |      | 공지 수정  | admin계정만       |
| DELETE |     |      | 공지 삭제  | admin계정만       |

### News (/api/news)
유료회원 무료회원에 따라 다르게 

| 메서드 | URL | BODY | 내용  | Security 허용 범위  |
| --- | --- | ---- | --- | --------------- |
| GET |     |      | 뉴스  | JWT있는지에 따라 다르게  |
