
#### 세팅 
- 테이블 명 : students
- 컬럼 : id, grade, name 
- 데이터 수 : 1천만명 
```SQL
CREATE TABLE students
(
    id INTEGER GENERATED ALWAYS AS IDENTITY,
    name VARCHAR(10),
    grade INTEGER CHECK (grade >= 0 AND grade <= 100)
)
```
```SQL 
INSERT INTO students(grade, name)
SELECT 
    Floor(random() * 100),
    '학생' || LPAD(n::text, 8, '0')
FROM generate_series(1, 10000000) AS s(n)
```


### 기본 조회 분석
> 데이터베이스가 이 행을 가져오기 위해 무엇을 했는지 파악하는데 집중하기 

#### 전체 조회 
```SQL 
EXPLAIN ANALYZE SELECT * FROM students

QUERY PLAN

Seq Scan on students  (cost=0.00..212501.70 rows=13897170 width=15) (actual time=7.018..1725.139 rows=10000000 loops=1)
Planning Time: 0.382 ms
JIT:
  Functions: 2
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 0.231 ms (Deform 0.113 ms), Inlining 0.000 ms, Optimization 0.496 ms, Emission 6.378 ms, Total 7.105 ms
Execution Time: 2078.589 ms
```
- 실행 시간 : 2078.589ms (매우 느리다)


#### ID 조건 값 조회 
```SQL 
EXPLAIN ANALYZE SELECT * FROM students WHERE id = 123
```

```SQL
QUERY PLAN

Gather  (cost=1000.00..126613.85 rows=1 width=23) (actual time=366.689..374.091 rows=0 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on students  (cost=0.00..125613.75 rows=1 width=23) (actual time=337.189..337.189 rows=0 loops=3)
        Filter: (id = 123)
        Rows Removed by Filter: 3333333
Planning Time: 0.812 ms
JIT:
  Functions: 12
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 1.610 ms (Deform 0.625 ms), Inlining 0.000 ms, Optimization 1.387 ms, Emission 25.613 ms, Total 28.610 ms
  
Execution Time: 422.612 ms
```
- 조금 더 빨라지긴 했는데 여전히 많이 느리다.
- **느린 이유** 
	- `ID = 123`이라는 조건에 인덱싱이 되어 있지 않기 때문에, 데이터베이스는 자신이 아는 방법인 **병렬 순차 스캔(Parallel Sequence Scan)** 을 수행하기로 결정
	- 이는 Table Scan이며 테이블 자체로 이동하여 말 그대로 한 줄씩 검색하고, 각 행을 가져온 다음`ID`가 7인지 확인(ID = 7 행을 찾을때까지)


### ID 인덱스 생성 후 전체 컬럼 조회 
> B-트리 인덱스를 생성하여 검색 속도를 조금 더 빠르게 만들 것
```SQL 
CREATE INDEX id_index ON students(id)
```

#### WHERE 조건 있을 때 똑같이 실행 
```SQL 
SELECT * FROM grades WHERE ID = 7

Index Scan using id_index on students  (cost=0.43..8.45 rows=1 width=23) (actual time=0.031..0.031 rows=0 loops=1)
  Index Cond: (id = 123)
Planning Time: 0.634 ms
Execution Time: 0.089 ms
```
- 매우 매우 빨라졌다. (422.6 ➡ 0.09)
- 이것은 Index Scan이다 Cuz `ID_index`라는 인덱스를 사용하여 `ID = 7`을 검색했기 때문

#### WHERE 조건 없을 때 똑같이 실행 
> [!WARNING] WHERE 절이 없으면 전체를 조회하기에 여전히 느리다
```SQL
EXPLAIN ANALYZE SELECT id, name FROM students
```

```SQL 
QUERY PLAN

QUERY PLAN

Seq Scan on students  (cost=0.00..163696.15 rows=10000115 width=23) (actual time=0.089..1024.483 rows=10000000 loops=1)
Planning Time: 0.392 ms
Execution Time: 1478.668 ms
```
- Indexing을 생성했음에도 실행시간이 굉장히 느리다.
- 이것은 아직도 여전히 힙영영의 Table Scan이 필요하기 때문 


### Index-Only-Scan의 예시 
> ID 인덱스 생성 후 ID 컬럼만 조회 

예전에 썼던 글에 다 나왔던 내용

```SQL
EXPLAIN ANALYZE SELECT ID FROM students WHERE id = 123
```

```SQL 
QUERY PLAN

Index Only Scan using id_index on students  (cost=0.43..4.45 rows=1 width=4) (actual time=0.055..0.056 rows=0 loops=1)
  Index Cond: (id = 123)
  Heap Fetches: 0
Planning Time: 0.568 ms
Execution Time: 0.074 ms
```
- Index-Only-Scan을 사용한다. 
- 데이터베이스가 테이블로 이동하지 않아도 된다. 테이블로 갈 필요가 없었기 때문에 인덱스 온리 스캔. 
- 만약 이게 가능하다면 정말 빠를 것 

> [!missing]  But 상식적으로 ID를 찾으려고 ID를 기준으로 `SELECT`한다는 것은 비상식적이다.

>[!SUCCESS]  실제 상식적으로 실용적인 Index-Only-Scan 방법을 어떻게 할 지 알아볼 것 



### 실용적 Index-Only-Scan 
> Non-Key Columns 에 주목 

> [!TIP] 클라이언트가 ID를 기준으로 name을 자주 요청한다는 가정 

#### 세팅 1. include 인덱스 생성 
기존 Index 삭제 후 재 생성 
```SQL
DROP INDEX id_index 

CREATE INDEX id_idx ON grade(id) INCLUDE (name)
```
INCLUDE를 포함한 컬럼을 'Non-Key Column'이라고 한다.
위 예시에서:
- id : key column
- name : non-key column

#### 세팅 2. VACCUME ⭐ - **visibility map** 갱신 
- PostgreSQL은 MVCC 구조이기 때문에, **해당 테이블의 튜플(행)** 이 **인덱스만으로 실제 테이블에 접근하지 않아도 되는 상태인지**를 확인해야 한다.
- 이건 내부적으로 "visibility map"을 통해 확인된다.
- 그런데 **VACUUM** 또는 **autovacuum**이 제대로 돌지 않았거나,
- 최근에 **대량 삽입/수정/삭제**가 이루어졌다면, visibility map이 갱신되지 않아 Index Only Scan이 불가능할 수 있다. 
- 따라서 아래와 같이 필요 
```SQL
VACUUM ANALYZE students;
```

>[!tip] Visibility Map
>- PostgreSQL이 각 테이블 블록(페이지)의 **모든 튜플(행)이 "모두 visible"한지** 여부를 추적하는 **비트맵**
>- 역할 **Index Only Scan 시 테이블을 굳이 안 봐도 돼"** 를 알려주는 지도 역할

>[!tip] VACUUM ❓
>- PostgreSQL이 테이블의 **죽은 튜플(dead tuples)**을 정리하고, **visibility map을 갱신**하는 명령
>- `Index Only Scan`을 쓸 수 있는 조건 중 하나가 **visibility map이 최신 상태**이기 때문


#### 실행 

```SQL
                               Table "public.students"
 Column |         Type          | Collation | Nullable |           Default            
--------+-----------------------+-----------+----------+------------------------------
 id     | integer               |           | not null | generated always as identity
 grade  | integer               |           | not null | 
 name   | character varying(20) |           |          | 
Indexes:
    "id_index" btree (id) INCLUDE (name)
```

```SQL
EXPLAIN ANALYZE SELECT name FROM students WHERE id = 8

Index Only Scan using id_index on students  (cost=0.43..4.45 rows=1 width=15) (actual time=0.136..0.136 rows=0 loops=1)
  Index Cond: (id = 8)
  Heap Fetches: 0
Planning Time: 0.510 ms
Execution Time: 0.183 ms
```
- Index Only Scan

```SQL 
VACUUM ANALYZE students; << 일단 이거하고 

EXPLAIN ANALYZE SELECT id, name FROM students

QUERY PLAN

Seq Scan on students  (cost=0.00..173528.84 rows=9999884 width=19) (actual time=4.769..1120.245 rows=10000000 loops=1)
Planning Time: 0.418 ms
JIT:
  Functions: 2
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 0.220 ms (Deform 0.146 ms), Inlining 0.000 ms, Optimization 0.535 ms, Emission 4.078 ms, Total 4.833 ms
Execution Time: 1458.141 ms
```
❓id, name이 전부 Index에 있는데 Seq Scan이 발생한 이유 ❓
- Index Only Scan이 _가능_하더라도 플래너는 **비용**이 낮은 경로를 고른다.
- `WHERE` 없이 10 M 행 전체를 읽는 쿼리는 **연속적인 Seq Scan이 더 저렴**하다고 판단되기 때문에 Index Only Scan을 선택하지 않는 것이 정상 동작


### INCLUDE가 빛을 발하는 상황 

이전에 WHERE절 or 전체 조회 에서 INCLUDE가 딱히 효과가 안 보였다.
그럼 언제 효과가 있을까??

모르겠다....




> [!WARNING] 인덱스가 무조건 좋은것은 아니다
> - 인덱스를 위한 heap fetch가 필요
> - 메모리 필요

