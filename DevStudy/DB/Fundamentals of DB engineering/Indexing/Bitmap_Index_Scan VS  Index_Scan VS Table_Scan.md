
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
> 쿼리 최적화 고급 기술
> - 여러 보조 인덱스 or 조건에 의해 후보 레코드(페이지)를 먼저 추려낸 후, 
> - 그 페이지들만 힙에서 순차적으로 접근해 데이터를 가져오는 방식.

#### 개념 
- **비트맵 인덱스**라는 특수한 인덱스를 활용하는 방식
- 인덱스에서 조건을 만족하는 **레코드의 위치(page 번호)를 비트맵 형태**로 수집
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

Bit number = Page number

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
    - 인덱스 조건(g > 95)을 만족하는 TIP(Tuple ID)를 탐색
    - 해당 TIP가 해당하는 페이지 번호를 비트맵에 기록
    - `EXPLAIN` 출력에선 **`-> Bitmap Index Scan on idx_name`** 로 표시.
        
2.  **Bitmap Heap Scan**
    - **우선**, 비트가 켜진 블록(찾을 페이지)을 **순차 I/O**로 읽고 페이지를 가져온다(**랜덤 I/O 감소 효과)**
    - **그 다음**, 각 페이지에서 조건을 _Recheck_ 한다.
	    - Recheck는 그렇게 비용이 들지 않는다.
	    - 왜냐하면 페이지를 일단 가져왔기 때문에 그 row들이 메모리에 이미 올려져있음. (ex. shared buffer)
    - **Recheck 이유** 
	    - 비트맵은 페이지 단위 기록이므로, 해당 페이지 내 모든 row가 조건을 만족하지 않을 수 있음
	    - 실제 row를 다시 확인하는 단계 (이미 메모리에 올라왔기 때문에 비교적 저렴)
    
    *랜덤 I/O 감소 효과*
	- 필요 컬럼만 가져오므로, 비트맵 단계에서 이미 Covering Index 요건을 충족했다면 **읽기량이 추가로 줄어든다.**

#### 면접 답변 
>[!QUESTION] 비트맵 스캔이란 무엇이고, 어떻게 작동하나요?

비트맵 스캔(Bitmap Scan)**은 PostgreSQL에서 인덱스를 활용하는 방식 중 하나로, **선택도가 중간 정도인 조건에 대해 효율적으로 데이터를 읽기 위한 전략**입니다.
특히, **랜덤 I/O 비용이 높은 상황에서 전체 테이블 스캔을 하기엔 너무 비싸고, 일반 인덱스 스캔(Index Scan)으로는 행마다 디스크 접근이 너무 많은 경우**, 그 중간 지점에서 성능을 최적화하기 위해 사용됩니다.
조건으로 필터링한 뒤 필요한 페이지 번호만 순차접근하여 불필요한 접근 방지

여러 인덱스를 조합할 수 있기 때문에, 비트맵 스캔은 **Bitmap AND/OR** 연산을 활용해 `조건 A AND 조건 B`처럼 **서로 다른 인덱스를 조합하는 상황에서 강력한 성능 이점을 제공**

비트맵 스캔에다는 2단계 구조를 가지고 있습니다.
첫 번째 단계는, Bitmap Index Scan입니다.
- 이는 인덱스를 사용해 조건에 해당하는 레코드들이 **어느 페이지에 있는지**를 찾습니다. 
- 이때 행 단위가 아니라, 해당 행이 속한 **페이지 번호만 비트맵으로 저장**합니다.

두 번째 단계는, Bitmap Heap Scan입니다.
- 첫 번째 단계에서 쓰인 비트맵에 저장된 페이지들을 한 번씩 순차적으로 접근합니다.
- 해당 페이지를 메모리에 올린 뒤, 거기서 조건에 맞는 row인지 재검증(recheck)를 합니다.
- 이를 통해 필요한 컬럼 값만 추출합니다.




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

정리

| 조건               | 실행 계획                       |
| ---------------- | --------------------------- |
| 선택도 매우 낮음 (≈ 0%) | Index Scan (랜덤 I/O 허용)      |
| 선택도 중간 (≈ 5~20%) | **Bitmap Index Scan** → 효율적 |
| 선택도 높음 (> 20%)   | Sequential Scan (전 테이블 순회)  |

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

#### Index Scan 쿼리들 
**☑쿼리** 
```SQL 
EXPLAIN SELECT name FROM grades WHERE id = 1000;

QUERY PLAN

Index Scan using grades_pkey on grades  (cost=0.42..8.44 rows=1 width=14)
  Index Cond: (id = 1000)
```
- Index Scan을 사용한다 using PRIMARY KEY 
- name은 Heap영역의 Table에 있으니 어쨌든 DISK에 접근은 해야한다. 
- **Note** : INDEX와 HEAD은 다른 데이터 구조 


**☑쿼리** 
```SQL 
EXPLAIN ANALYZE SELECT name FROM grades WHERE id < 1000;

QUERY PLAN

Index Scan using grades_pkey on grades  (cost=0.42..44.69 rows=1044 width=14) (actual time=0.028..0.283 rows=999 loops=1)
  Index Cond: (id < 1000)
Planning Time: 0.227 ms
Execution Time: 0.352 ms
```
>1000건 모두 Index Scan → 힙 접근 1000번 → **랜덤 I/O 발생**
- 데이터베이스는 인덱스를 사용하여 `ID` 인덱스에 대해 Index Scan을 수행하고, 이 때 각 인덱스당 한번씩 힙 페이지에 접근 
- 그 페이지에는 하나 이상의 행이 있고, 거기서 해당 값을 가져온다. 
- 문제 : **랜덤 액세스(random access)** 발생 💢
	- 999개의 값에 대해 페이지로 가서 행을 가져왔다.
	- 999개의 모든 행에 대해 그렇게 했다.
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
- **조건: g > 95**은 되게 선택도가 높은 조건이다.
- 전체 100만 건 중 약 5만 건 정도가 매칭
- 선택도가 5%정도로 중간 정도이므로 Bitmap Index Scan을 사용한 것 같다.



**☑Bitmap Index Scan + Filter**
```SQL 
CREATE INDEX ON grades(g); -- g도 인덱스 생성 for bitmap

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
id, g 둘 다 인덱스인데 로그를 보면 g만 Bitmap Index Scan으로 사용된 것을 볼 수 있다.
❓두 개 다 Bitmap Index Scan에 사용하면 더 빠를 것 같은데 **왜 g만 인덱스로 활용** ❓
- PostgreSQL 옵티마이저는 **id의 선택도가 매우 낮기 때문에** 인덱스로 활용하지 않았다.
- 지금 데이터가 100만개인데 id > 5000 이면 99만 5천개의 데이터가 저 조건에 포함이다.
- 따라서 옵티마이저는 선택도가 높은 g만 Bitmap Index Scan을 활용하고 페이지에 접근한 뒤 거기서 id 필터링을 하는 최적화 방식을 선택했다.

>선택도 : 조건이 얼마나 **좁은지**를 나타냄




>만약 위의 Bitmap Index Scan + Filter 예시에서 WHERE절에서 ID가 아닌 name이였다면 선택도가 높아져서 name, g 모두 Bitmap Index Scan에 쓰였을 것이다.


