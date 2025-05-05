
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
from pymongo import MongoClient
client = MongoClient('localhost', 27017)
db = client.jungle

  
# 여러개 찾기 find() : 조건에 맞는 모든 문서를 가져온다.
# db.users.find({}) : users 컬렉션에서 모든 문서를 가져온다.
all_users = list(db.users.find({})) # {} 조건 없이 모두
print(all_users)
print(all_users[0])
print(all_users[0]['name'])
print(all_users[0]['age'])

  
# find_one() : 조건에 맞는 첫 번째 문서만 가져온다.
user = db.users.find_one({'name': 'John'})
print(user)

# find_one의 두 번째 인자 : 프로젝션 => 어떤 필드를 포함하고 제외할지를 결정
user2 = db.users.find_one({'name': 'John'},{'_id': False, 'name': False})
print(user2)


# 수정하기
# db.people.update_many(찾을조건,{ '$set': 어떻게바꿀지 })
db.users.update_many({'name': 'Carrel'}, {'$set': {'age': 2222}})
carrel = db.users.find_one({'name': 'Carrel'})
print(f"{carrel['name']}의 나이는 {carrel['age']}입니다.")

# 삭제하기
db.users.delete_one({'name': 'Carrel'})
```



