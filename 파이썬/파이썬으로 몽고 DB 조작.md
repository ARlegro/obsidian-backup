
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



### 영화 스크래핑 후 DB 저장 

>[!SUCCESS]  미션 
>영화 랭킹 사이트가서 title, 출시연도, 러닝타임, 등급 스크래핑 후 MongoDB에 저장 


```python
import requests
from bs4 import BeautifulSoup
from pymongo import MongoClient


header = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36"
}

client = MongoClient("localhost", 27017)
db = client.movie_db

def make_movie_doc():

    url = "https://www.imdb.com/chart/top/?ref_=nv_mv_250"
    res = requests.get(url, headers=header)
    soup = BeautifulSoup(res.text, "lxml")
    movies = soup.select(".ipc-metadata-list .cli-parent .cli-children")
    print("movies 개수 : ", len(movies))

    for movie in movies:
        title = movie.select_one("h3").text
        metadatas = movie.select(".cli-title-metadata-item")
        released_year = metadatas[0].text
        running_time = metadatas[1].text
        pg_level = metadatas[2].text

        db.movies.insert_one(
           {
                "title": title,
                "released_year": released_year,
                "running_time": running_time,
                "pg_level": pg_level,
            }
        )

make_movie_doc()

all_movies = db.movies.find({})  # Cursor 객체를 반환
print(all_movies)  # 그냥 print하면 Cursor 객체의 주소값이 나옴
print(type(all_movies))  # <type 'pymongo.syncronous.cursor.Cursor'>
print(list(all_movies))  # Cursor 객체를 list로 변환하여 출력 or for문으로 돌리기

```

