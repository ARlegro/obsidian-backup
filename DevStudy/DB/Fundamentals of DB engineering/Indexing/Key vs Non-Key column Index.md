

### 인덱스 기본개념 : B-Tree와 검색 원리 
데이터베이스에서 인덱스를 생성할 때는 하나 이상의 필드(컬럼)를 지정하여 그 위에 **B-Tree**라는 자료구조를 만듭니다. 이 필드(들)가 **키(Key)**가 되며, 검색 목적으로 사용



인덱스를 통해 검색을 수행하면, 데이터베이스 플래너는 이 인덱스를 활용하여 원하는 데이터를 찾습니다. 인덱스에서 발견된 항목들은 실제 테이블의 레코드를 가리키는 **행 포인터(ROWID)** 를 포함하고 있습니다. 따라서 인덱스를 통해 특정 레코드를 찾으면, 해당 ROWID를 사용하여 테이블에서 실제 데이터를 가져올 수 있다

### Non-Key Included Index 
모든 데이터베이스에 존재하는 기능은 아니지만, **PostgreSQL**에는 **논키(Non-Key) 포함(Included) 인덱스**라는 강력한 기능이 있습니다. 이는 인덱스를 생성할 때 검색 키로 사용되지 않는 추가 컬럼들을 인덱스 내에 **포함(Include)**시킬 수 있는 기능

예를 들어, `grades` 필드에 인덱스를 생성하면서 `ID` 필드를 포함(Include)할 수 있습니다. 이렇게 하면 `grades` 필드를 기준으로 검색하고 `ID`를 선택하는 쿼리를 실행할 때, 데이터베이스는 더 이상 실제 테이블(힙)로 이동할 필요가 없습니다. 필요한 모든 정보(검색 키인 `grades`와 포함된 `ID`)가 이미 인덱스 내에 존재하기 때문

이러한 논키 포함 인덱스는 **인덱스 온리 스캔(Index Only Scan)** 이라는 최적화된 실행 계획을 가능하게 하여 상당한 성능 향상을 가져올 수 있다.

## 실습 

#### 세팅 

1. **students 테이블에 500만 데이터 저장** 
	- 이떄 정말 많은 컬럼을 넣어볼 것 (아무거나 id5, id7, id8) 
	- Cuz 특정 컬럼만 가져오는 비용 비교해봐야하니까
```SQL
CREATE TABLE students (
    id        INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    grade     INT NOT NULL,
    firstname VARCHAR(5),
    lastname  VARCHAR(20),
    id6  INT, id7  INT, id8  INT, id9  INT,
    id10 INT, id11 INT, id12 INT, id13 INT
);

INSERT INTO students (grade, firstname, lastname, id6, id7, id8, id9)
SELECT floor(random()*100), '학생', n::text,
       (random()*1000)::INT, (random()*1000)::INT,
       (random()*1000)::INT, (random()*1000)::INT
FROM generate_series(1, 5_000_000) AS s(n);
```

2. **`VACUUM` 실행:**
	```SQL
	VACUUM ANALYZE students;           -- 가시성 맵 + 통계 갱신	
	```
	 - 최적의 쿼리 성능을 위해 `VACUUM` 명령을 실행하여 가시성 맵(visibility map)을 포함한 모든 통계 정보를 최신 상태로 업데이트
	 - `VACUUM VERBOSE students` 명령을 사용하여 자세한 정보를 확인
	 - **`VACUUM`의 중요성:** PostgreSQL에서 Index_Only_Scan이 제대로 작동하려면 `VACUUM`을 통해 가시성 map이 최신 상태로 유지되어야 한다. 그렇지 않으면 인덱스만으로도 충분한 정보가 있더라도 불필요하게 힙 fetch가 발생할 수 있다.

### 1. 인덱스 없이
```SQL
EXPLAIN ANALYZE SELECT ID, grade
FROM students
WHERE grade > 80 AND grade < 95
ORDER BY grade DESC;
```
- 이거는 **정말 많은 row에 접근해야하기** 때문에 꽤 **비싼 쿼리**이다.
- 따라서 비추 쿼리💢 
- 아래의 쿼리 플랜을 보자 

```SQL 
QUERY PLAN

Gather Merge  (cost=108924.92..175899.53 rows=574028 width=8) (actual time=326.958..457.742 rows=699845 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Sort  (cost=107924.90..108642.43 rows=287014 width=8) (actual time=299.499..323.693 rows=233282 loops=3)
        Sort Key: grade DESC
        Sort Method: external merge  Disk: 4464kB
        Worker 0:  Sort Method: external merge  Disk: 4032kB
        Worker 1:  Sort Method: external merge  Disk: 3888kB
        ->  Parallel Seq Scan on students  (cost=0.00..77978.99 rows=287014 width=8) (actual time=8.163..222.713 rows=233282 loops=3)
              Filter: ((grade > 80) AND (grade < 95))
              Rows Removed by Filter: 1433385   

Planning Time: 0.524 ms
JIT:
  Functions: 12
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 1.412 ms (Deform 0.305 ms), Inlining 0.000 ms, Optimization 1.398 ms, Emission 22.886 ms, Total 25.696 ms
Execution Time: 526.393 ms
```
- **실행 시간** : 526.39 ms
- `ORDER BY` : Sort작업이 있다.
- `Parallel Seq Scan` : 아직 grade에 대한 인덱스가 없기에 이 스캔을 통해 **힙에 있는 데이터가 포함된 페이지를 가져옴** 
- `Rows Removed by filter`
	- **힙에서 해당 페이지를 가져와 메모리(shared buffer)로 올라온 뒤**, **튜플 단위**로 `WHERE` 절을 평가하여 필터링한다.

  
>이러한 비효율성 떄문에 인덱스가 필요 


### 2. 인덱스 없이 LIMIT 절 추가 
```SQL
EXPLAIN ANALYZE 
SELECT ID, grade 
FROM students 
WHERE grade > 80 AND grade < 95 
ORDER BY grade DESC 
LIMIT 1000;


QUERY PLAN

Limit  (cost=94715.68..94832.35 rows=1000 width=8) (actual time=298.741..303.809 rows=1000 loops=1)
  ->  Gather Merge  (cost=94715.68..161690.29 rows=574028 width=8) (actual time=298.547..303.528 rows=1000 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Sort  (cost=93715.66..94433.19 rows=287014 width=8) (actual time=288.017..288.069 rows=1000 loops=3)
              Sort Key: grade DESC
              Sort Method: top-N heapsort  Memory: 113kB
              Worker 0:  Sort Method: top-N heapsort  Memory: 113kB
              Worker 1:  Sort Method: top-N heapsort  Memory: 114kB
              ->  Parallel Seq Scan on students  (cost=0.00..77978.99 rows=287014 width=8) (actual time=0.090..253.444 rows=233282 loops=3)
                    Filter: ((grade > 80) AND (grade < 95))
                    Rows Removed by Filter: 1433385
Planning Time: 0.891 ms
Execution Time: 304.131 ms
```
시간이 줄긴했다. (줄어든 이유는 지금 시도하려는 것과 다른 주제라 PASS)
`LIMIT`를 걸었음에도 불구하고 순차 스캔은 전체를 검색해야 하므로 크게 빨라지지 않는다.


>[!tip] 줄어든 이유 간단 분석 - top-N heapsort 
>- 각 워커가 “상위 1,000개”만 유지하도록 힙 구조로 정렬  
>- 전체 정렬 대신 heap push/pop 연산으로 비용 절감
>- **`LIMIT 1000` 조건 덕분에, 내부적으로 “1,000개 이상의 값은 heap에서 즉시 버림” 처리**

### 3. 단순 B-Tree 인덱스 
grade 필드에 B-Tree 인덱스 생성하고 테스트해볼 것 
```SQL
CREATE INDEX ON students (grade);
```
이 인덱스는 `grade` 필드를 정렬된 형태로 저장하여 검색 및 정렬 작업을 효율적으로 만든다.

확인 (\d students)
```SQL
                                Table "public.students"
  Column   |         Type          | Collation | Nullable |           Default            
-----------+-----------------------+-----------+----------+-----------------------
 id        | integer               |           | not null | generated always as identity
 grade     | integer               |           | not null | 
 
	....
 id13      | integer               |           |          | 
Indexes:
    "students_grade_idx" btree (grade)
```
- grade 컬럼의 btree 인덱스가 생겼따.


#### 동일한 쿼리 재실행 (Without LIMIT)
```SQL
EXPLAIN ANALYZE
SELECT ID, grade
FROM students
WHERE grade > 80 AND grade < 95
ORDER BY grade DESC;

QUERY PLAN                                                                   

Sort  (cost=142672.39..144394.48 rows=688833 width=8) (actual time=581.272..642.640 rows=699845 loops=1)
  Sort Key: grade DESC
  Sort Method: external merge  Disk: 12384kB
  ->  Bitmap Heap Scan on students  (cost=9396.97..66458.47 rows=688833 width=8) (actual time=46.168..425.100 rows=699845 loops=1)
        Recheck Cond: ((grade > 80) AND (grade < 95))
        Heap Blocks: exact=46729
        ->  Bitmap Index Scan on students_grade_idx  (cost=0.00..9224.76 rows=688833 width=0) (actual time=33.383..33.384 rows=699845 loops=1)
              Index Cond: ((grade > 80) AND (grade < 95))
Planning Time: 0.497 ms
JIT:
  Functions: 4
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 0.214 ms (Deform 0.065 ms), Inlining 0.000 ms, Optimization 0.524 ms, Emission 3.470 ms, Total 4.208 ms
  
Execution Time: 690.382 ms
```
> Flow : Bitmap Index Scan → Bitmap Heap Scan → Sort

**결과 분석:**
- **실행 시간:** 더 느려졌다.
- **인덱스 스캔 후 테이블 접근:** 
	- 인덱스를 사용하여 `grade` 조건을 만족하는 레코드를 찾는다. 
	- 하지만 쿼리는 `ID` 컬럼도 요청하고 있는데, `ID`는 인덱스에 포함되어 있지 않고 있다. 
	- 따라서 데이터베이스는 인덱스에서 ROWID를 얻은 후, **해당 ROWID를 사용하여 다시 테이블(힙)로 이동하여 `ID` 값을 가져와야 한다.** 이 테이블 접근(힙 페치)이 대부분의 비용을 차지

#### 동일한 쿼리 재실행 (With LIMIT) ⭐⭐

```SQL 
EXPLAIN ANALYZE SELECT ID, grade
FROM students
WHERE grade > 80 AND grade < 95
ORDER BY grade DESC
LIMIT 1000;

Limit  (cost=0.43..295.17 rows=1000 width=8) (actual time=0.118..1.553 rows=1000 loops=1)
  ->  Index Scan Backward using students_grade_idx on students  (cost=0.43..203027.42 rows=688833 width=8) (actual time=0.102..1.479 rows=1000 loops=1)
        Index Cond: ((grade > 80) AND (grade < 95))
Planning Time: 0.656 ms
Execution Time: 1.603 ms
```
> Flow : Index Scan Backward using students_grade_idx
- **실행 시간:** 1.6ms로 훨씬 빨라졌다. 이는 `LIMIT`가 적용되어 **Index Scan이 필요한 최소한의 레코드만 찾고 테이블로 접근하기 때문**입니다.
- `LIMIT` 덕분에 인덱스의 ‘Stop-after-N’ 성질이 발휘된다.

### Covering Index (Non-Key Included Index)⭐⭐⭐

기존 인덱스를 삭제하고 `grade` 필드에 인덱스를 생성하되, `ID` 필드를 포함(Include)시킨다.
```SQL
DROP INDEX students_grade_idx; -- 기존 인덱스 삭제
CREATE INDEX ON students (grade) INCLUDE (ID);
```

이 인덱스는 `grade`를 기준으로 정렬되며, 각 `grade` 항목에 해당 레코드의 `ID` 값도 함께 저장
물론, 이로 인해 인덱스 크기는 더 커진다.

```SQL
                            Table "public.students"
  Column   |         Type          | Collation | Nullable |           Default    
-----------+-----------------------+-----------+----------+-----------------------
 ...
 grade     | integer               |           | not null | 
	...
 id13      | integer               |           |          | 
Indexes:
    "students_grade_id_idx" btree (grade) INCLUDE (id)
```

####  동일한 쿼리 재실행 without `LIMIT`
```sql
EXPLAIN ANALYZE 
SELECT ID, grade
FROM students
WHERE grade > 80 AND grade < 95
ORDER BY grade DESC;

// 인덱스 온리 스캔 
Index Only Scan Backward using students_grade_id_idx on students  (cost=0.43..21345.09 rows=688833 width=8) (actual time=0.028..87.644 rows=699845 loops=1)
  Index Cond: ((grade > 80) AND (grade < 95))
  Heap Fetches: 0
Planning Time: 0.082 ms
Execution Time: 108.971 ms
```
> Index Only Scan

**결과 분석:**
- **실행 시간:** 108ms (이전 기본 : 526ms, 단순 인덱스 : 690 ms)
- **인덱스 온리 스캔(Index Only Scan):** 가장 중요한 변화는 `Index Only Scan`이 발생했다는 점입니다. 쿼리에서 요청하는 `ID`와 `grade` 컬럼이 모두 인덱스 내에 존재하므로, **테이블(힙)로의 접근(`Heap Fetches`)이 0**입니다. 이는 디스크 I/O 비용을 크게 줄여준다.
- **디스크 I/O:** 인덱스 자체도 디스크에 저장되어 있으므로, 인덱스 크기가 커서 메모리에 완전히 캐시되지 않으면 여전히 디스크에서 인덱스 페이지를 읽어와야 한다. 그럼에도 불구하고 테이블 전체를 읽는 것보다 훨씬 효율적


#### 동일한 쿼리 재실행 with `LIMIT`

```SQL 
EXPLAIN ANALYZE 
SELECT ID, grade
FROM students
WHERE grade > 80 AND grade < 95
ORDER BY grade DESC
LIMIT 1000;

Limit  (cost=0.43..31.42 rows=1000 width=8) (actual time=0.068..0.361 rows=1000 loops=1)
  ->  Index Only Scan Backward using students_grade_id_idx on students  (cost=0.43..21345.09 rows=688833 width=8) (actual time=0.066..0.287 rows=1000 loops=1)
        Index Cond: ((grade > 80) AND (grade < 95))
        Heap Fetches: 0
Planning Time: 0.668 ms
Execution Time: 0.483 ms
```
**실행 시간:** 0.5밀리초(ms) 미만으로, 이전 B-Tree 인덱스에 `LIMIT`를 적용했을 때(1.6ms)보다도 더 빨라졌다.

### 요약 

>[!danger] '인덱스 없는 순차 스캔'은 모든 데이터를 읽고 필터링하므로, **대량의 불필요한 I/O가 발생하며 매우 느리다.**  
1. **B-Tree 인덱스 스캔** 
	- 인덱스를 사용하여 검색 조건에 맞는 ROWID를 빠르게 찾지만,  ✅
	- **요청된 컬럼이 인덱스에 없으면** 다시 테이블로 이동(**Heap Fetch**)해야 하므로 여전히 **I/O 오버헤드가 발생**합니다. 💢
	  
2. **논키(Non-Key) 포함(Included) 인덱스 (인덱스 온리 스캔)**
	- 쿼리에 필요한 모든 컬럼이 인덱스 내에 존재하여 테이블로의 접근이 전혀 필요 없다. 
	- 이로 인해,  **I/O 비용을 극적으로 줄여 가장 빠른 성능을 제공**


### 고려 사항 및 결론
- **인덱스 크기:** 논키 컬럼을 포함하면 인덱스 크기가 커지므로, 인덱스 생성 시간이 길어지고, 더 많은 디스크 공간을 차지하며, 메모리에 완전히 캐시되지 못할 가능성이 높아집니다.
- **사용 사례:** 논키 포함 인덱스는 **테이블 크기가 매우 크고, 특정 컬럼 조합에 대한 쿼리가 빈번하며, 해당 쿼리가 인덱스에 포함된 컬럼만으로도 결과를 반환할 수 있을 때** 가장 큰 성능 이점을 제공합니다.
- **`VACUUM`의 중요성:** PostgreSQL에서 인덱스 온리 스캔이 제대로 작동하려면 `VACUUM`을 통해 가시성 맵이 최신 상태로 유지되어야 합니다. 그렇지 않으면 인덱스만으로도 충분한 정보가 있더라도 불필요하게 힙 페치가 발생할 수 있습니다.
- **옵티마이저의 역할:** 데이터베이스 옵티마이저는 통계 정보와 쿼리 형태를 기반으로 어떤 인덱스를 사용할지, 심지어 인덱스를 사용할지 여부를 결정합니다. 따라서 인덱스를 생성한다고 항상 사용되는 것은 아닙니다.