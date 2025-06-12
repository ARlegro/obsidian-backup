
https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS

두 트랜잭션이 동일한 행에 대해 충돌하는 락을 동시에 가질 수 없다.
행 레벨 락은 데이터 쿼리(읽기)에는 영향을 미치지 않으며, **동일한 행에 대한 쓰기(writers)와 잠금(lockers) 작업만 차단**

### 1. FOR UPDATE - 완전 배타락 
- `SELECT` 문으로 검색된 행들을 마치 업데이트를 위한 것처럼 잠근다.
- 이는 현재 트랜잭션이 끝날 때까지 다른 트랜잭션이 이 행들을 잠그거나, 수정하거나, 삭제하는 것을 방지

### 2. FOR NO KEY UPDATE - 부분 배타락 
*PostgreSQL 9.3 이후 버전부터 제공
- `FOR UPDATE`와 유사하게 동작하지만, 획득하는 락이 좀 더 약하다.
- **행(row) 자체의 상태 변경(UPDATE/DELETE)만 막고, 외래 키→참조 무결성(foriegn key constraint)을 확인하기 위한 조회만은 허용**한다.
- 즉, 완전한 FOR UPDATE 보다는 **충돌 범위를 좁힌 것

### 3. FOR SHARE - 공유락 
- `FOR NO KEY UPDATE`와 유사하게 동작하지만, 검색된 각 행에 대해 **배타적 락(exclusive lock) 대신 공유 락(shared lock)**을 획득
- **차단하는 것 ⛔**
	- 다른 트랜잭션이 이 행들에 대해 `UPDATE`, `DELETE`, `SELECT FOR UPDATE` 또는 `SELECT FOR NO KEY UPDATE`를 수행하는 것을 차단
- **차단하지 않는 것** ✔
	- `SELECT FOR SHARE` 또는 `SELECT FOR KEY SHARE`를 수행하는 것은 막지 않는다.

### 4. FOR KEY SHARE - 키 공유락 
- `FOR SHARE`와 유사하게 동작하지만, 락이 더 약하다
- **변경분** : `SELECT FOR UPDATE`는 차단되지만, `SELECT FOR NO KEY UPDATE`는 차단되지 않는다
- 그니까, 다른 트랜잭션이 그 Row를 DELETE하거나 키 값을 변경하는 것을 차단하지만, 다른 `UPDATE`(ex. id제외 필드값 변경)는 허용하는 것 
- 예시:
	- FK를 가진 테이블에 데이터를 삽입할 때, 참조된 부모 테이블의 해당 행에 대해 **`FOR KEY SHARE`** 잠금

> 공유락 : 테이블에 새로운 행을 넣는 동안 부모 행이 삭제되거나 기본 키 값이 변경되는 것을 막아, 참조 무결성을 보장

## 참고 :  **Conflicting Row-Level Locks**
이 표는 현재 획득된 락 모드와 요청된 락 모드 간의 충돌 여부를 보여준다.
X = 충돌 = 요청된 락이 현재 획득된 락에 의해 차단됨

| 세로 : 요청 락 모드<br>가로 : 현재 락 모드 | Current Lock Mode |             |                     |              |
| ---------------------------- | :---------------: | :---------: | :-----------------: | :----------: |
| `Requested Lock Mode`        |  `FOR KEY SHARE`  | `FOR SHARE` | `FOR NO KEY UPDATE` | `FOR UPDATE` |
| `FOR KEY SHARE`              |                   |             |                     |      X       |
| `FOR SHARE`                  |                   |             |          X          |      X       |
| `FOR NO KEY UPDATE`          |                   |      X      |          X          |      X       |
| `FOR UPDATE`                 |         X         |      X      |          X          |      X       |
