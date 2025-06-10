> PostgreSQL 관점 

Postgre는 무슨 Query Plans을 짤까❓❓

### 📖Query Plan 
- **목적** : SQL 쿼리를 가장 효율적인 실행 방식으로 변환
- **방법**:
    - 여러 **실행 계획 시나리오 생성**  
    - 각 시나리오의 **예상 비용(cost)** 계산
    - 가장 비용이 낮은 계획을 선택
- **비용은 절대적인 시간이 아니라 상대적 비교 지표**  
	- PostgreSQL은 I/O, CPU 사용 등을 내부적으로 모델링하여 추정

>이때 선택된 실행 계획이 **EXPLAIN** 명령으로 확인할 수 있는 **Query Plan**


>[!tip] EXPLAIN  &  EXPLAIN ANALYZE 
>|명령어|설명|
|---|---|
|`EXPLAIN`|실행 계획만 보여줌 (추정치 기반)|
|`EXPLAIN ANALYZE`|쿼리를 실제 실행하고, 실행 시간과 실제 결과도 같이 제공|



### Query Plan 해설 
EXPLAIN 다음에 쿼리문을 작성하면 Query Plan이 나온다
```SQL 
postgres=# EXPLAIN select * from employees;
```

```SQL
                        QUERY PLAN                         
-----------------------------------------------------------
 Seq Scan on employees  (cost=0.00..16370.00 rows=1000000 width=18) (actual time=0.070..50.091 rows=1000000 loops=1)
 Planning Time: 0.059 ms
 Execution Time: 74.229 ms
(3 rows)
```

💚**Seq Scan**
- 전체 테이블을 순차적으로 탐색 (*Full-Table-Scan*)
- 굉장히 **비효율적** 쿼리 Cuz 힙에 접근하고 모든 테이블에 접근하려하기 때문 

💚**cost=0.00 .. 16370.00** 
- PostgreSQL이 예측한 비용(상대적 수치)
- 두개의 점 (..)을 기준으로 나뉜다.
- **첫 분할 숫자(0.00) 뜻**
	- 첫 row를 얻는 데(첫 페이지를 가져오는데) 걸리는 비용 (빠르면 0)
	- 이게 0이 아닐 경우 : 데이터를 가져오기 전에 몇 가지 작업을 하는 것 
	  
- **두번째 분할 숫자 뜻** 
	- 전체 쿼리 실행 시 예상 총 비용

💚**rows** 
- Planner가 예측한 결과 rows 수 
- 실제 수는 아니다 

💚**width** 
- 각 row의 평균 바이트 크기 
- 디스크 I/O, 메모리 사용량 계산 시 중요 기준이 됨
- 이거를 줄여야 network비용을 줄일 수 있음 ⭐
	- 이래서 `TEXT` 말고 `VARCHAR(10)`이런거 쓰나 봄 


ROW의 개수를 대략 파악하기에는 `EXPLAIN`이 좋다. 
- `SELECT COUNT`함수는 성능을 매우 저하시킨다
- 그래서 `EXPLAIN`으로 대량 얼마나 row있는지 확인 


## ORDER BY의 영향 실험 

#### 세팅 
Grade 테이블에 2천만개의 데이터 존재 
```SQL 
INSERT INTO grades(name)
SELECT 
    LPAD ((floor(random() * 10000)::int)::text, 5, '0')
FROM generate_series(1, 20000000) 
```

인덱스 키 없는 테이블을 전체 조회한다했을 때 planner는비용이 많이 들거라고 예상했다.

#### 기본 EXPLAIN
```SQL
EXPLAIN SELECT * FROM grades 

QUERY PLAN
Seq Scan on grades  (cost=0.00..260178.24 rows=17168224 width=14)
```
2000만개의 데이터임에도 불구하고 Planner의 예상 rows는 다르다.


### ORDER BY가 미치는 영향 ⭐⭐
```SQL 
EXPLAIN SELECT * FROM grades ORDER BY name;

QUERY PLAN

Gather Merge  (cost=1358641.53..3303231.00 rows=16666746 width=6)
  Workers Planned: 2
  ->  Sort  (cost=1357641.51..1378474.94 rows=8333373 width=6)
        Sort Key: name		        
        ->  Parallel Seq Scan on grades  (cost=0.00..171829.73 rows=8333373 
width=6)
```
> 순서 : Parallel Seq Scan ➡ Sort ➡ Merge Sort

비용만 봐도 엄청 오래 걸릴 것으로 Planner가 예상했다.

Sort + Gather Merge
- Gather Merge ➡ 정렬 유지 상태로 데이터를 병합 (병합 정렬)
- Sort ➡ 병렬로 처리되는 worker들이 각자 맡은 부분의 데이터를 가져와서 데이터를 정렬한다.
- Worker들이 정렬한 데이터들을 병합정렬 시킨다. 

> 흐름 정리 : worker별 스캔 ➡ worker별 스캔데이터 정렬 ➡ 전역 병합 정렬 

**비싼 이유 분석** : `ORDER BY name` 때문에 파이프라인에 **Sort + Gather Merge**가 삽입됐고, 이것이 비용을 급격히 높였다.


**❓ORDER BY가 플랜을 어떻게 바꿨나**
일단, Planner는 인덱스가 없다면 순차 스캔 후 전역 병합 정렬이 가장 저렴하다고 판단했다.
그래서 아래와 같이 예상 플랜을 바꾼 것 

| 조건                                 | 예상 플랜                                                            |
| ---------------------------------- | ---------------------------------------------------------------- |
| **정렬 없음** (`SELECT * FROM grades`) | `Gather → Parallel Seq Scan` (오직 테이블 스캔 후 스트림 병합)                |
| **정렬 있음** (`ORDER BY name`)        | `Gather Merge → Sort → Parallel Seq Scan` (스캔한 뒤 워커별 정렬 + 전역 병합) |

>[!tip] 빈번히 같은 정렬을 요구한다면, **B-tree 인덱스 구축**이 가장 손쉬운 개선


### ORDER BY 에서 인덱싱 최적화 

```SQL
CREATE INDEX idx_grades_name ON grades(name);
```

```sql
SELECT * FROM grades ORDER BY name;
```
쿼리에서는 정렬 없이도 정렬된 결과를 인덱스로 가져올 수 있다:



### 추가 실험 

```SQL 
EXPLAIN SELECT * FROM grades where id = 10;

QUERY PLAN

Index Scan using grades_pkey on grades  (cost=0.44..8.46 rows=1 width=10)
  Index Cond: (id = 10)
```
- 초기 Cost(0.44) : 인덱스 스캔을 위해 heap에 jump한다?? 그래서 드는 비용 

**✅width 줄이기 = 네트워크 비용 감소**
- `SELECT id`만 쓰면 width 줄어든다.
- `TEXT` 대신 `VARCHAR(10)` 쓰는 이유도 여기에 있음   
- 서버 → 클라이언트 전송 시 속도 향상


### 실전 팁 

| 주의할 점                                             |
| ------------------------------------------------- |
| **`ORDER BY` 빈번히 쓰이는 컬럼에 인덱스가 없으면 성능 폭망**         |
| **`LIKE '%abc%'` 는 인덱스 무효화** (Full Table Scan 유발) |
| VACUUM이 되어야 Index Only Scan이 가능할 수도 있음            |
| `EXPLAIN`만으로는 실제 쿼리 성능 파악 어려움 → `ANALYZE` 필수      |
