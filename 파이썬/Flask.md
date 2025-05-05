
>[!SUCCESS]  파이썬 웹 프레임워크  ➡ 서버 구동시 필요한 여러 기능 제공 

적은 노력으로 많은 작업을 할 수 있는 도구 

> [!INFO] 설치 
> `pip install flask`
> 


### 개념 




### 실행

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

