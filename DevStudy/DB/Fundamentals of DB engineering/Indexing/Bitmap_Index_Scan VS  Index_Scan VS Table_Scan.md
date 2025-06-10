
> DB에서 데이터를 읽어오는 방식은 크게 3가지로 나뉠 수 있다.
> 효율적인 데이터 접근은 쿼리 성능에 직접적인 영향을 미치므로 각 스캔 방식의 이해는 매우 중요


## 개념 정리 

### 1. Index Scan 

#### 개념 
- DB의 인덱스를 활용하여 원하는 데이터를 찾아내는 방식
- 특정 값이 저장된 레코드의 물리적 위치(Row_ID, Page)를 갖고 있다.

#### 원리 
1. **인덱스 검색** : 쿼리 조건에 해당하는 값을 인덱스에서 검색 
2. **ROWID 획득** : 해당 값이 매핑된 데이터 레코드의 물리적 주소(ROWID)획득
3. **테이블 접근** : 2번에서 획득한 ROWID를 사용하여 실제 테이블에 접근해서 해당 레코드를 직접 찾아 읽어옴 

#### 장단점 
**✅장점** 
- **빠른 속도** 
	- 적은 양의 데이터를 검색할 때 매우 빠르다
	- 이유 : 인덱스가 특정 칼럼에 대해 정렬되어 있고, 실제 데이터를 직접 찾아가기 때문
	  
- **부분 범위 처리** 
	- 범위 데이터 검색 시 인덱스의 정렬된 특성을 활용하여 효율적으로 처리

**💢 단점**
- **추가 저장 공간 必** 
	- 인덱스 자체를 유지하고 관리하는 데 추가적인 저장 공간 필요 
- **오버헤드**
	- DML(INSERT, UPDATE, DELETE) 작업 시 오버헤드가 발생
- **랜덤 I/O**로 오히려 속도가 더 느릴 수도 



### 2. Table Scan
aka Full-Table-Scan, Seq Scan
#### 개념 
- 인덱스를 사용하지 않고, 테이블에 있는 모든 데이터를 처음부터 끝까지 읽어들이는 방식

#### 작동 방식 
- **Full Scan** : 데이터베이스가 해당 테이블이 저장된 디스크 블록을 처음부터 순차적으로 읽어나갑니다.
- **조건 검사** : 읽어온 각 레코드에 대해 쿼리의 조건(WHERE 절)을 하나씩 비교하며 만족하는 레코드만 필터링

#### 장단점
**✅장점** 
- 간단함 
- 테이블 대부분을 스캔해야하는 경우에는 오히려 Table Scan이 효율적일 수 있음 
	- 심지어 병렬 쿼리 시 테이블 스캔 빠름 

**💢단점** 
- 특정 소량의 데이터를 찾을 때 매우 비효율
	- 필요한 레코드가 몇 개 되지 않더라도 전체 테이블을 다 읽어야 하기 때문

### 3. Bitmap Index Scan 
> 여러 보조 인덱스를 활용하여 쿼리를 최적화하는 고급 기술

#### 개념 
- **비트맵 인덱스**라는 특수한 인덱스를 활용하는 방식
- 여러 인덱스 조건을 AND/OR로 결합해 “TID 비트맵”을 만든다. 
- **카디널리티(Cardinality)가 낮은 칼럼**, 즉 중복되는 값이 많고 고정된 몇 가지 값만 가지는 칼럼(예: 성별, 주중/주말, Yes/No 등)에 효과적
#### 작동 방식
1. **비트맵 생성**
	- 여러 인덱스를 개별적으로 스캔하여 해당 조건을 만족하는 행들이 위치한 `페이지의 비트맵`을 생성
	- 이 비트맵에서 각 비트는 테이블의 레코드 하나를 나타내며, 해당 레코드가 특정 값을 가지면 '1', 아니면 '0'으로 표시
	  
2. **비트맵 연산**
	- 쿼리 조건에 따라 여러 비트맵을 비트 OR, AND, NOT 연산 등을 통해 결합
3. **ROWID 획득** 
	- 연산된 최종 비트맵에서 '1'로 표시된 위치(비트)에 해당하는 레코드의 ROWID들을 추출
4. **테이블 접근** 
	- 추출된 ROWID들을 사용하여 실제 테이블에서 해당 레코드를 읽어

> 이 최종 비트맵을 사용하여 힙(Heap)으로 **한 번만 점프하여 필요한 페이지들을 효율적으로 가져**온다. 

#### 장단점 

**✅장점** 
- **낮은 카디널리티에 효율적** 
	- 중복 값이 많은 컬럼에 대해서는 B-Tree 인덱스보다 더 효율적 공간사용 및 성능 good
- **다중 컬럼 조건에 강점**
	- 여러 비트맵 인덱스 칼럼에 대한 `AND`, `OR` 조건을 매우 빠르게 처리할 수 있다.
- **랜덤 I/O 줄어든다**

> **적합한 경우:** 중복 값이 많은 칼럼에 대한 쿼리가 빈번하고, 데이터 변경이 적은 분석 시스템

**💢단점** 
- DML에 취약
	- 데이터 변경 작업 시 비트맵 전체를 갱신해야 할 수 있기에 성능저하 발생 가능 

#### Bitmap Index Scan → Bitmap Heap Scan : 단계별 내부 흐름

1. **Bitmap Index Scan**
    - TID를 **비트맵(생략 가능 = lossy, 정확 = exact)** 구조에 기록.
    - 여러 인덱스를 쓴다면 비트맵 간 AND/OR 연산으로 결합하여 최종 후보 블록 집합을 만든다.
    - `EXPLAIN` 출력에선 **`-> Bitmap Index Scan on idx_name`** 로 표시.
        
2.  **Bitmap Heap Scan**
    - 비트가 켜진 블록을 **순차 I/O**로 읽고, 블록 안에서 조건을 _Recheck_ 한다.
    - **랜덤 I/O 감소 효과**
	    - 필요 컬럼만 가져오므로, 비트맵 단계에서 이미 Covering Index 요건을 충족했다면 **읽기량이 추가로 줄어든다.**

--- 

## 비교 
비트맵 인덱스 스캔은 매우, 매우 영리한 메커니즘 

테이블을 어떻게 쿼리하여 특정 데이터를 어떻게 가져오는지를 알아볼 것 
실제 쿼리를 실행한다기보다는 `EXPLAIN`을 사용하여 설명할 것 
- 실제 성능보다는, 이 쿼리 실행 시 내부적으로 PostgreSQL이 무엇을 하려는지 알고 싶기 때문 

### 1. 옵티마이저가 'Bitmap Scan'을 고르는 조건 

> **인덱스 하나로는 행 수가 많아 랜덤 I/O가 과해지고, ‘전체 테이블 스캔’(Seq Scan)까지 갈 정도는 아닌”** 회색 지대에서 등장
- 예상 행 비율이 5~20% 부근이면 Index Scan보다 Bitmap Scan이 이득이라고 판단
- 랜덤 페이지 I/O 비용 많이 설정 시 : 랜덤 블록 접근이 비싸게 설정될수록, ‘먼저 비트맵으로 후보 페이지만 모아 순차 I/O’ 전략을 선호

```TEXT
          WHERE 절 집합
                 │
      ┌──────────┴──────────┐
      │ 선택도 극소 (≈0 %)  │ 선택도 중간(≈5–20 %)
      │      └─> Index Scan │      └─> Bitmap Path
      │                     │
선택도 높음(>20 %) ─────────┘
            └─> Seq Scan
```



### 2. 예시 
#### 세팅 
```SQL 
CREATE TABLE grades 
(
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT,
    g INTEGER,
    firstname TEXT,
    lastname TEXT,
    address TEXT,
    bio TEXT
)

\d grades: 

          Table "public.grades"
  Column   |  Type   | Collation | Nullable |           Default            
-----------+---------+-----------+----------+------------------------------
 id        | integer |           | not null | generated always as identity
 name      | text    |           |          | 
 g         | integer |           |          | 
 firstname | text    |           |          | 
 lastname  | text    |           |          | 
 address   | text    |           |          | 
 bio       | text    |           |          | 
Indexes:
    "grades_pkey" PRIMARY KEY, btree (id)

```
영상에서는 g필드에 b-tree인덱스를 넣었다 
`"g" btree(g) INCLUDE(id)`
```SQL
INSERT INTO grades(name, g, firstname, lastname)
SELECT 
    '학생' || LPAD(Floor(random()*n)::text, 7, '0'),
    Floor(random()*100),
    '김이박',
    'Real Name' || LPAD(n::text, 7, '0')
    
FROM generate_series(1, 1_000_000) AS s(n)
```

#### 인덱스 스캔 쿼리들 
☑쿼리 
```SQL 
EXPLAIN SELECT name FROM grades WHERE id = 1000;

QUERY PLAN

Index Scan using grades_pkey on grades  (cost=0.42..8.44 rows=1 width=14)
  Index Cond: (id = 1000)
```
- Index Scan을 사용한다 using PRIMARY KEY 
- name은 Heap영역의 Table에 있으니 어쨌든 DISK에 접근은 해야한다. 
- Not : INDEX와 HEAD은 다른 데이터 구조 


☑쿼리 
```SQL 
EXPLAIN ANALYZE SELECT name FROM grades WHERE id < 1000;

QUERY PLAN

Index Scan using grades_pkey on grades  (cost=0.42..44.69 rows=1044 width=14) (actual time=0.028..0.283 rows=999 loops=1)
  Index Cond: (id < 1000)
Planning Time: 0.227 ms
Execution Time: 0.352 ms
```
- 이것도 Index Scan 사용 
- 데이터베이스는 인덱스를 사용하여 `ID` 인덱스에 대해 인덱스 스캔을 수행하고, 하나씩 스캔하면서 100 미만인 모든 것을 찾기로 결정한다. 
- B-트리에서 이는 매우 빠르게 수행된다. 
- 결과적으로 인덱스를 사용. 
- 그리고 찾은 각 값에 대해 해당 행이 존재하는 페이지를 찾고, 힙으로 돌아가서 그 페이지를 가져온다.
- 그 페이지에는 하나 이상의 행이 있고, 거기서 해당 값을 가져옵니다. 
- 이것을 **랜덤 액세스(random access)** 라고 부른다 💢
	- 99개의 값에 대해 페이지로 가서 힙으로 가서 행을 가져왔습니다.
	- 99개의 모든 행에 대해 그렇게 했습니다.
	- 따라서 많은 행이 있다면 이것이 느려질 수 있다


#### Bitmap Scan 쿼리 

`Note : 0 <= g <= 100`
```SQL 
CREATE INDEX ON grades(g);

EXPLAIN SELECT name FROM grades WEHRE g> 95;

Bitmap Heap Scan on grades  (cost=467.34..11298.18 rows=41667 width=14) (actual time=3.376..22.746 rows=39852 loops=1)
  Recheck Cond: (g > 95)
  Heap Blocks: exact=10111
  ->  Bitmap Index Scan on grades_g_idx  (cost=0.00..456.93 rows=41667 width=0) (actual time=2.040..2.040 rows=39852 loops=1)
        Index Cond: (g > 95)
Planning Time: 0.050 ms
Execution Time: 23.885 ms
```

왜 Index Scan이 아닌 Bitmap Index Scan을 썼을까?

- 페이지를 찾아도 바로 테이블로 가지 않는다.
- 대신, set a bit?
- 예를 들어, page 9에 있는 row를 찾았다고 하자
	- 그때 table로 가는게 아니라 Bitmap의 9번자리를 1로 채운다.
	- 결과적으로, 정확히 가져와야 할 페이지 수를 알고 있기 때문에, 힙으로 한 번 점프하여 이 모든 페이지를 가져와야 합니다. 그리고 이 모든 페이지를 가져오면 어떻게 될까요? 각 페이지에는 하나 이상의 행이 있을 것입니다. 그렇죠? Postgres가 데이터를 저장하는 방식입니다.
	- 그런데, 그 행들 중 일부는 `G > 95` 기준을 만족하지 않을 수도 있습니다. 그래서 Postgres는 조건을 **재확인(recheck)**하여 필터링된 행들을 제외하고, 필터링되지 않은 행들만 남깁니다. 이 경우, 운 좋게도 정확한 수의 행이 페이지에 정확히 들어맞았습니다. 그렇죠? 그것이 비트맵 인덱스 스캔

Bitmap Scan에서 
bit number = page number


☑Bitmap and Scan 

```SQL 
EXPLAIN ANALYZE SELECT name FROM grades WHERE g > 95 AND id > 5000

QUERY PLAN

Bitmap Heap Scan on grades  (cost=467.29..11402.29 rows=41446 width=14) (actual time=3.647..21.783 rows=39666 loops=1)
  Recheck Cond: (g > 95)
  Filter: (id > 5000)
  Rows Removed by Filter: 186
  Heap Blocks: exact=10111
  ->  Bitmap Index Scan on grades_g_idx  (cost=0.00..456.93 rows=41667 width=0) (actual time=2.318..2.319 rows=39852 loops=1)
        Index Cond: (g > 95)
Planning Time: 0.091 ms
Execution Time: 23.209 ms
```


- bitmap이 2개인데 2개의 bitmap에 공통된 페이지만 가져오면 돼서 굿이다.
- 이렇게 공통된 페이지만 가져오게 마지막에는 Bitmap Heap Scan을 한다
- 이것 또한 recheck를 한다.
	- recheck는 그렇게 비용이 들지 않는다. 왜냐하면 페이지를 일단 가져왔기 때문에 그 row들이 메모리에 이미 올려져있음. (ex. shared buffer)



