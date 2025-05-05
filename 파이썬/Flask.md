
>[!SUCCESS]  파이썬 웹 프레임워크  ➡ 서버 구동시 필요한 여러 기능 제공 

적은 노력으로 많은 작업을 할 수 있는 도구 

> [!INFO] 설치 
> `pip install flask`
> 


### 개념
✔






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


