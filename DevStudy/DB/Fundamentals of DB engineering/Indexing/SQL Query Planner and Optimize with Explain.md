> PostgreSQL 관점 


Postgre는 무슨 Query Plans을 짤까❓❓


EXPLAIN 다음에 쿼리문을 작성하면 Query Plan이 나온다

```SQL 
postgres=# EXPLAIN select * from employees;
                        QUERY PLAN                         
-----------------------------------------------------------
 Seq Scan on employees  (cost=0.00..16370.00 rows=1000000 width=18)
(1 row)

--------------------------------------------------
 Seq Scan on employees  (cost=0.00..16370.00 rows=1000000 width=18) (actual time=0.070..50.091 rows=1000000 loops=1)
 Planning Time: 0.059 ms
 Execution Time: 74.229 ms
(3 rows)
```

Seq Scan
- 순차적 스캔 = Full-Table-Scan이랑 거의 같다 
- 굉장히 비효율적 쿼리 Cuz 힙에 접근하고 모든 테이블에 접근하려하기 때문 

cost=0.00 .. 16370.00 
- PostgreSQL이 예측한 상대 비용
- 두개의 점 (..)을 기준으로 나뉜다.
- **첫 분할 숫자(0.00) 뜻**
	- 첫 페이지를 가져오는데 걸린 시간 (ms)
	- 0인 이유 : Postgres는 즉시 테이블에가서 첫 로우를 가져와서 결과를 반환?
	- 그래서 첫 결과의 비용은 거의 없는 것 
	- 이게 0이 아닐 경우 : 데이터를 가져오기 전에 몇가지 작업을 하는 것 
	  
- **두번째 분할 숫자 뜻** 
	- 총 걸리는 예상 시간 

rows 
- Planner가 예측한 결과 rows 수 
- 실제 수는 아니다 

width 
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

> 흐름 정리 : 워커별 스캔 ➡ 워커별 스캔데이터 정렬 ➡ 전역 병합 정렬 

**비싼 이유 분석** 
- `ORDER BY name` 때문에 파이프라인에 **Sort + Gather Merge**가 삽입됐고, 이것이 비용을 급격히 높였다.


**❓ORDER BY가 플랜을 어떻게 바꿨나**
일단, Planner는 인덱스가 없다면 순차 스캔 후 전역 병합 정렬이 가장 저렴하다고 판단했다.
그래서 아래와 같이 예상 플랜을 바꾼 것 

| 조건                                 | 예상 플랜                                                            |
| ---------------------------------- | ---------------------------------------------------------------- |
| **정렬 없음** (`SELECT * FROM grades`) | `Gather → Parallel Seq Scan` (오직 테이블 스캔 후 스트림 병합)                |
| **정렬 있음** (`ORDER BY name`)        | `Gather Merge → Sort → Parallel Seq Scan` (스캔한 뒤 워커별 정렬 + 전역 병합) |

>[!tip] 빈번히 같은 정렬을 요구한다면, **B-tree 인덱스 구축**이 가장 손쉬운 개선


### 추가 실험 

```SQL 
EXPLAIN SELECT * FROM grades where id = 10;

QUERY PLAN

Index Scan using grades_pkey on grades  (cost=0.44..8.46 rows=1 width=10)
  Index Cond: (id = 10)
```
- 초기 Cost(0.44) : 인덱스 스캔을 위해 heap에 jump한다?? 그래서 드는 비용 
- 
