

### 인덱스란 ❓
> 데이터베이스 테이블에 대한 검색 성능을 높여주는 자료 구조 
> #데이터구조 #효율적_검색
- 기존 테이블 위에 구축하고 할당하는 데이터 구조
- 테이블을 살펴보고 분석하여 일종의 **지름길**을 만드는 역할
- 실생활 예시 : 사전에 label 붙어 있는 것 
- 흔한 인덱스 유형으로는 B-Tree, LMS trees가 있다. (이거는 나중에)

모든 기본 키(Primary Key)는 기본적으로 인덱스를 가진다. 이는 B-tree 인덱스???



### 성능 비교하기 전 세팅 : 인덱싱 O or X 

#### 기본 세팅 및 분석 
테이블 생성 후 테이블 정보를 볼 것이다.
```SQL
CREATE TABLE employees
(
    id serial PRIMARY KEY,
    name VARCHAR(10)
)

#psql /d employees

                                  Table "public.employees"
 Column |         Type          | Collation | Nullable |                Default                
--------+-----------------------+-----------+----------+---------------------------------------
 id     | integer               |           | not null | nextval('employees_id_seq'::regclass)
 name   | character varying(10) |           |          | 
Indexes:
    "employees_pkey" PRIMARY KEY, btree (id)

```
- 인덱스를 설정하지 않았는데, 기본적으로 인덱스가 설정되어 있다.
	- 이유 : **모든 Primary Key는** 기본적으로 **b-tree 인덱스**를 가지고 있다 ⭐

#### 더미 데이터 - 100만 
```SQL 
INSERT INTO employees(name)
SELECT 
    '직원' || LPAD(n::text, 7, '0')
FROM generate_series(1, 1000000) AS s(n)
```

```SQL
SELECT * FROM employees LIMIT 10;
id,name
1,직원0000001
2,직원0000002
3,직원0000003
4,직원0000004
5,직원0000005
6,직원0000006
7,직원0000007
8,직원0000008
9,직원0000009
10,직원0000010
```


#### EXPLAIN ANALYZE 도입 
쿼리 성능을 비교하기 위해 **`EXPLAIN ANALYZE`** 를 사용할 것
- _쿼리를 실제로 실행한 뒤에 그 결과를 통해 실행 계획을 보여준다_.
- 실행할 쿼리 설명해주고, 시간이 얼마나 걸렸는지 알려준다.


아래는 테스트 결과이다. 
```SQL
-- 모든 column
EXPLAIN ANALYZE SELECT * FROM employees WHERE id = 1;

-- id만 가져오기 
EXPLAIN ANALYZE SELECT id FROM employees WHERE id = 1;

-- ----
QUERY PLAN

Index Scan using employees_pkey on employees  (cost=0.15..8.17 rows=1 width=42) (actual time=0.027..0.027 rows=0 loops=1)
  Index Cond: (id = 1)
Planning Time: 0.633 ms
Execution Time: 0.058 ms

------
Index Only Scan using employees_pkey on employees  (cost=0.15..8.17 rows=1 width=4) (actual time=0.013..0.014 rows=0 loops=1)
  Index Cond: (id = 1)
  Heap Fetches: 0
Planning Time: 0.488 ms
Execution Time: 0.092 ms

```
- 100만개의 데이터를 전부 뒤지는게 아니다.
- 자동으로 인덱싱이 일어났다. 
	- B-Tree 인덱스를 가진 PRIMARY KEY를 활용 ➡ 빨라짐 

☑`Heap Fetches` 
두번째 id 컬럼만 가져오는 명령문의 `EXPLAIN ANALYZE`를 보면 `Heap Fetches = 0`이 있다.
- **이유** : 쿼리한 값, 즉 **ID는 인덱스에 있었기 때문에** 이 정보를 가져오기 위해 **힙(heap)으로 갈 필요가 없었다**. ID가 인덱스에 있었기 때문입니다. 그래서 그냥 인라인으로 가져온 것 
- 이를 **"인라인(inline) 쿼리"** 라고 함 

> 만약 인라인 쿼리가 가능하다면 접근 비용이 비싼 Heap을 사용하지 않아도 돼서 굉장히 빨라질 것 

**☑Planning time** 
- 쿼리를 계획
- **인덱스를 사용할지**❓아니면 **테이블을 스캔할지**❓ 결정하는 것

**☑Execution Time**
- 실제 작업을 수행하는 것


>[!QUESTION] 인덱스를 가져오는거 자체도 시간이 걸리지 않을까 ❓
>- 일반적으로 실제 테이블이 인덱스보다 훨씬 크다.
>- 인덱스는 하나의 데이터 구조이고, 테이블은 다른 데이터 구조
>- 테이블이 실제로 더 더 무겁다 ❗❗
>- 따라서, 테이블에 가는 것을 최대한 피하려고한다.


## 본격 실험 

> [!success] 시나리오를 나눌 것 (이를 통해 비용 비교할 것)
> 1. ID만 수정 
> 2. name 만 수정 
> 3. name 조건으로 ID 조회 




### 시나리오 1. Index Only Scan
> ID만 사용해서 조회 

```SQL
EXPLAIN ANALYZE SELECT id FROM employees WHERE id = 5000;
```

```SQL
 Index Only Scan using employees_pkey on employees  (cost=0.42..4.44 rows=1 width=4) (actual time=0.912..0.913 rows=1 loops=1)
   Index Cond: (id = 5000)
   Heap Fetches: 0
 Planning Time: 4.492 ms
 Execution Time: 0.988 ms
(5 rows)
```
Index Only Scan 
- 일단 Heap영역에 접근하지 않았다는 것을 보면 빠를 것이라고 예상할 수 있다.
- Execution Time은 0.027ms 로 매우 빠르다.

> 자세한 결과 내용 분석은 `EXPLAIN ANALYZE 도입`부분에서 했으니 Pass 




### 시나리오 2. name 조회하기 

#### 실행 
```SQL 
EXPLAIN ANALYZE SELECT name FROM employees WHERE id = 6000;

Index Scan using employees_pkey on employees  (cost=0.42..8.44 rows=1 width=14) (actual time=0.816..0.819 rows=1 loops=1)
	 Index Cond: (id = 6000)
Planning Time: 1.237 ms
Execution Time: 0.998 ms
(4 rows)
```
엥❓별 차이 없는데???. 위의 결과는 SQL Client를 썼을 경우의 결과이다.

혹시나 시나리오 1번에서 캐시된 것일수도 있기에 다른 id 로 재시도 
```SQL

Index Scan using employees_pkey on employees  (cost=0.42..8.44 rows=1 width=14) (actual time=6.077..6.083 rows=1 loops=1)
   Index Cond: (id = 12000)
Planning Time: 0.090 ms
Execution Time: 6.115 ms
(4 rows)
```
❗매우 느려졌다 6.115ms (이전에는 0.988)❗

#### 느린 이유 분석
- 인덱스에서 ID를 찾았지만, name컬럼을 얻기위해서는 **디스크에 있는 테이블 행으로 점프해야 했기 때문**
 - Index에는 `name` 컬럼이 없고, **Heap-Table에 있다**. ⭐⭐⭐
 - 그렇기 떄문에 실제로 검색하기 위해서는 순차적으로 employees 테이블을 Scan해야 함 (최악의 경우 full-table-scan)
- 이것은 가능한 한 피해야 한다. 💢 

>[!tip] PostgreSQL의 순차 스캔 최적화 
>- Postgres는 여러 스레드, 즉 워커 스레드를 실행하여 **병렬로 순차 스캔**을 수행함으로써 좀 더 영리하게 행동하려고 함.
>- 그래서 실제 PostgreSQL은 **이론보다는 좀 더 빠른 순차 스캔이 가능** 
>- **그래도 피하려고 노력해봐야 함** 


>[!danger] 어찌됐던 Heap 접근을 피하려고 노력해라. 그 다음은 Full-Table-Scan피하려고 노력


### 시나리오 3. name을 WHERE 절에 사용 🔐
>[!danger] 우선, 이 CASE는 굉장히 오래걸릴 것이다.

없는 이름을 조건으로 넣어서 쿼리를 짤 것. 이렇게 해야 Full-Tabel-Scan이 무조건 일어나므로 

```SQL 
EXPLAIN ANALYZE SELECT id FROM employees WHERE name = 'szzzz';
```

```SQL 
Gather  (cost=1000.00..12578.43 rows=1 width=4) (actual time=101.348..103.515 rows=0 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on employees  (cost=0.00..11578.33 rows=1 width=4) (actual time=94.829..94.830 rows=0 loops=3)
         Filter: ((name)::text = 'szzzz'::text)
         Rows Removed by Filter: 333333
Planning Time: 0.652 ms
Execution Time: 103.542 ms  <<<< 
(8 rows)
```
- 걸린 시간 : 103 ms 💢
- 매우 매우 느리다.
- **이유 분석**
	1. name 컬럼은 Index가 없다.
	2. 즉, name조건에 맞는 데이터를 찾기 위해서는 테이블을 하나하나 전부 찾아야 함 

*Note : PostgreSQL은  Parallel Seq Scan으로 내부 최적화가 있긴해서 이론보다 더 빠를 것*


### LIKE 쿼리가 안 좋은 이유 
`LIKE`는 가장 나쁜 쿼리문 중에 하나이다 
- 모든 행을 통과하면서 일치 여부를 확인해야 하기 때문 
- '시나리오 3'같은 Full-Table-Scan이 발생




--- 
## 인덱스 도입 

### 인덱스 만들기 

```SQL
CREATE INDEX 인덱스명 ON 테이블명(컬럼명)
-- 보통 인덱스명은 '테이블명_컬럼명'으로 한다.
```

참고로 인덱스 생성하는 데 시간 좀 걸린다 Cuz B-Tree를 만들어야하므로

이전에 느렸던 (시나리오 3) 같은 쿼리로 테스트하기 

```SQL
EXPLAIN ANALYZE SELECT id, name FROM employees WHERE name = 'DF';
```

```SQL
 Index Scan using employees_name on employees  (cost=0.42..8.44 rows=1 width=18) (actual time=0.820..0.845 rows=0 loops=1)
   Index Cond: ((name)::text = 'DF'::text)
 Planning Time: 2.700 ms
 Execution Time: 0.872 ms
(4 rows)
```
- **실행 시간이 엄청 줄었다.** (103.54ms -> 0.87) ⭐
- 대신 Planning Time이 조금 늘음
- 이유 분석 
	1. Index로 인해 더 적은 Row를 읽어도 된다.
	2. 심지어, id-name은 전부 인덱스로 되어있기에 금방 찾는다.


>[!tip] 가끔 Bitmap Index Scan이 발생할 수 있다.
>- 이건 DB가 알아서 정하는 것 


### 만약 인덱스 + Like를 쓴다면?
>[!danger] 쓰지마라
>이거는 Index로도 해결하지 못 할 정도로 느린 쿼리이다.
>LIKE쿼리는 Index로 Scan이 불가능하다.
>즉, 최적화가 힘든게 LIKE 쿼리이다.
>비싸기만 함 
```SQL 
EXPLAIN ANALYZE SELECT id, name FROM employees WHERE name LIKE '%deFhzc%'
```
- %~%는 Single Value가 아니다.

>

