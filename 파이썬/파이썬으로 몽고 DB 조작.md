
MongoDB라는 프로그램을 조작하기 위해서는 특별한 라이브러리인 `pymongo`가 필요 
`pip install pymongo`



### 기본 - 객체 생성, 서버 연결, Insert
```python
from pymongo import MongoClient
# MongoClient는 MongoDB 서버와 연결하는 클라이언트 객체를 생성하는 클래스

client = MongoClient('localhost', 27017)
# MongoDB 서버에 연결.
# localhost는 기본값이며, 27017은 MongoDB의 기본 포트

db = client.jungle
# jungle이라는 DB를 만든다.
  
# users라는 컬렉션에 {'name': 'John', 'age': 30}이라는 문서를 추가한다.
db.users.insert_one({'name': 'John', 'age': 30})
db.users.insert_one({'name': 'Charles', 'age': 26})
db.users.insert_one({'name': 'Carrel', 'age': 23})

all_users = list(db.users.find({}))
print(all_users)
```


### 여러 메서드 - find, update
```python

```
