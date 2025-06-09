
데이터를 아무런 순서 없이 저장하는 구조로, 데이터가 입력 순서대로 디스크에 저장됨 (정렬 없음)
`HEAP`은 **정렬되지 않은 테이블 구조**로, 보통 **클러스터링 인덱스(Clustered Index)가 없는 테이블**을 의미

PostgreSQL은 기본적으로 **모든 테이블을 힙(Heap)** 으로 관리

```SQL
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(10)
);
```
- 이 테이블은 **Heap 구조**로 만들어지며, `id`가 PRIMARY KEY이더라도 물리적으로 `id`순으로 저장되지는 않는다.
### 힙 vs 인덱스 조직 테이블 (Clustered Table)

| 구분               | 힙 (Heap Table)         | 클러스터형 인덱스 테이블         |
| ---------------- | ---------------------- | --------------------- |
| 정렬               | 없음                     | 인덱스 순서로 정렬됨           |
| INSERT 성능        | 빠름                     | 느릴 수 있음 (정렬 비용)       |
| SELECT 성능        | 느림 (풀스캔 필요)            | 빠름 (인덱스 정렬 유지)        |
| UPDATE, DELETE 후 | 공간 비효율 가능 (Dead Tuple) | 정리 효과 있음 (B-tree 유지됨) |

> Heap 테이블의 장점은 빠른 INSERT !!! 
> 만약 읽기 성능이 중요할 경우 
> - 인덱스 활용 
> - CLUSTER 활용 등을 고려해야 함 