
>[!SUCCESS]  파이썬 웹 프레임워크  ➡ 서버 구동시 필요한 여러 기능 제공 



> [!INFO] 설치 
> `pip install flask`
> 


### 개념
> 적은 노력으로 많은 작업(웹 페이지, API 서버 구축)을 할 수 있는 도구 

**✔마이크로 프레임워크이다.**
- 필수 기능만 제공 ex. 라우팅, 렌더링 등 
	- 추가로 필요한 기능이 있으면 그 때 확장 `pip install`
- 자유도가 매우 높다

**✔ 주요 클래스 - 뒤의 preview 참고** 

| 용어              | 설명               |
| --------------- | ---------------- |
| Flask           | 웹 서버를 만들기 위한 클래스 |
| @app.route()    | URL 경로를 지정하는 것   |
| run()           | 서버 실행 메서드        |
| request         | 요청 처리            |
| render_template | HTML 템플릿 렌더링용 도구 |



### preview - 간단한 실행
#### 기본 
```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

app.run()
```
파이썬은 라우터 설정하고 run() 흠 일단 ok 

파이썬의 기본 포트 = 5000
만약 바꾸고 싶다면 `app.run(port=5001)`

#### 디버깅 모드로 실행시키기 
일반적으로 서버를 실행시키고 코드를 바꾸면 바로 반영이 안된다.
이럴 때, 파이썬에서는 **디버그 모드**(스프링에서는 devtools였나??)로 설정하면 **코드 변경이 실행중인 서버에 반영**된다. 
```python
app.run(debug=True)
```


### 라우팅 
- 네트워크에서 경로를 선택하는 프로세스
- Client **요청 URL을 기준으로 어떤 함수를 실행할지를 결정하는 매핑**(mapping) 규칙 
- 즉, 어떤 URL로 접근했을 때 ➡ 어떤 함수로 연결할지를 정해주는 역할 
- 이로 인해, URL 별 다양한 처리 방식 가능

> 일종의, URL 과 함수를 연결짓는 매개체

> [!INFO] 라우터 
>URL과 함수를 연결해주는 것 
>Flask는 기본적으로 라우터 객체가 따로 없고 @app.route()로 직접 경로를 지정 
>But 규모가 커지면 Blueprint라는 것을 쓴다내


Flask는 기능도 몇 개 없어서 라우팅 할 줄 알면 Flask의 절반은 안 것 


