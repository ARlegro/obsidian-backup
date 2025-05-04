>[!SUCCESS]  파이썬 라이브러리 중 하나이다 
### 개념 
- HTML, XML **파일을 파싱**하고, 원하는 **데이터를 쉽게 추출**할 수 있도록 도와주는 파이썬 라이브러리 
- 구조화된 마크업 언어(HTML/XML)를 **파이썬 객체로 변환**하여 JS의 DOM탐색처럼 **노드에 접근**할 수 있게 한다.

### 설치 및 import
```pip install bs4```
```import bs4```

### BeautifulSoup 객체 생성 
```python
import bs4

html = "<html><body></body><html>"
bs4.BeautifulSoup(html, features)  # features는 그냥 인자 이름 
```
**features**
- HTML 구조를 Text로 바꾼 것을 bs4가 처리할텐데, 그 스트럭쳐를 해석하는 feature를 선택 
- 다양한 feature(parser)들이 존재하는데 
- **종류** 
	- lxml : 매우 빠름. 설치 필요 `pip install lxml`
	- html.parser : 파이썬 내장, 속도는 느리지만 설치 불필요
	- html5lib : HTML5 규격에 맞게 파싱. 매우 관대함. 느림 

### select() 활용 
배워보니까 CSS의 Selector기반 노드 접근 방법이다.
#### 기본 
```python
soup.select("ul.total") # ul테그 중 class가 total인 요소 
# 단순 ul 하면 여러 값이 나오므로 이렇게 필터링 가능 

soup.select("ul.total li") # 위 + 자식 li 요소를 가져오기 
```
- 이렇게 select해서 가져온 데이터의 타입 = ResultSet이라는 클래스 형태 
	`<class 'bs4.element.ResultSet'>`

#### 중첩 테그에 select()
```python
soup.select("ul.total li")
```
이렇게 가져온 HTML의 테그들에서 추가로 select를 쓸 수 있다.

```python
li_elements = soup.select("ul.total")
ul_element.select("li")
```
이렇게 하면 특정 요소, tag 내에서 select()하게 되는 것 

#### 한 번에 select()
```python
li_elements = soup.select("ul.total li")
```
이렇게 한 번에 select 할 수 있다.

> [!WARNING] **한번에 select()문 주의** 
> **⚠️1. ul.total li 이 여러개일 경우** 
> - 만약 `ul`테그의 `class` 이름이 `total`인 요소가 여러개라면 여러개가 합쳐져서 `ResultSet`을 반환한다
> 
> **⚠️직계 자손 li가 또 다른 li 자식을 갖고 있을 경우**
> - 위처럼 하면 그 li요소도 포함된다.
> - 해결 방법 ( > 표시 )
> ```python
>li_elements = soup.select("ul.total > li")
>```
>이렇게 하면 직게 자손의 li 테그만 select()


#### 특정 속성 조건으로 필터링

```python
soup.select("a[href^='https']")  # href 가 https로 시작하는 a테그 
```



## 실전

>[!QUESTION] 문제
>https://www.imdb.com/chart/top/?ref_=nv_mv_250 에서 첫 번째 영화의 제목, 개봉 연도, 상영시간, 영상물 등급도 같이 스크래핑


### 내 정답 
```python
import requests, lxml
from bs4 import BeautifulSoup


header = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36"
}

res = requests.get("https://www.imdb.com/chart/top/?ref_=nv_mv_250", headers=header)

  

data = BeautifulSoup(res.text, "lxml")
movies = data.select(".cli-children")
target_movie = movies[0]
name = target_movie.select_one("h3").text
meta_datas_spans = target_movie.select("div.bnnHxo > span")

year = meta_datas_spans[0].text
time = meta_datas_spans[1].text
grade = meta_datas_spans[2].text


print("제목 : ", name)
print("개봉 연도 : ", year)
print("상영 시간 : ", time)
print("영상물 등급 : ", grade)
  

# 첫 번째 영화의 제목, 개봉 연도, 상영시간, 영상물 등급도 같이 스크래핑
```

### 클프 정답 
```python
import requests
from bs4 import BeautifulSoup

headers = {'User-Agent' : 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36'}
data = requests.get('https://www.imdb.com/chart/top/?ref_=nv_mv_250', headers=headers)

soup = BeautifulSoup(data.text, 'html.parser')
movies = soup.select('.cli-parent')

# 첫번째 영화 요소 추출
movie = movies[0]

# 제목 요소 추출
h3_element = movie.select_one('h3')
print(h3_element.text)

# 개봉 연도 요소 추출
year_element = movie.select('.cli-title-metadata-item')[0]
print(year_element.text)

# 상영 시간 요소 추출
runtime_element = movie.select('.cli-title-metadata-item')[1]
print(runtime_element.text)

# 등급 요소 추출
rating_element = movie.select('.cli-title-metadata-item')[2]
print(rating_element.text)

```

