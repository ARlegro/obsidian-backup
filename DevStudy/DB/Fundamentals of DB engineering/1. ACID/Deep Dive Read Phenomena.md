[[2. Isolation of ACID]] << 여기서 언급 했었음 


## 1.  Phantom Reads
>[!tip] 한 트랜잭션 내 같은 퀄리 두 번 실행했을 때, 첫 번째 쿼리에서는 없던 **'new row'가 두 번째 쿼리에서 나타나는** 현상 
>- 보통 동시성이 있고, 트랜잭션 격리 수준이 충분히 높지 않으면 발생한다.


많은 사람들이 간과하는 개념이다. 잘못 사용하면 엄청 큰 문제를 일으킨다.

### 문제가 되는 예시 



```SQL
INSERT INTO salses (id, price, date) 
VALUES (1, 15, 'feb-7-2024');
```

### 해결 in Postgres
postgres는 Phantom Read를 해결하기 위한 방법을 제공한다.

>[!tip] PostgresSQL은 Repeatble Read 격리 수준임에도 Phantom Read를 방지한다.
>- **반면**, MySQL, Oracle, SQL Server같은 다른 DB에서는 
>	1. **Serializable** 격리 수준을 사용해야 Phantom Read를 제거할 수 있다.
>	2. 또는, **비관적 락**을 사용해서 팬텀 리드 방지 

#### MVCC for protecting Phantom Read 
Postgres가 낮은 격리 수준에도 Phantom Read를 방지할 수 있는 이유는 '**MVCC**'구현 덕분이다.
- Postgres는 MVCC(Multi-Version Concurrency Control) 버전관리 방식으로 트랜잭션을 처리한다. 
- **트랜잭션이 시작될 때 특정 시점의 데이터 스냅샷을 생성**하며, 이 스냅샷은 트랜잭션 수명 동안 유지됩니다.  ⭐⭐
- 이 스냅샷은 기존 행의 변경뿐만 아니라 **새로운 행의 삽입까지도 해당 트랜잭션에게는 보이지 않도록 처리**

추가로, 이러한 방법은 범위 잠금이 필요가 없는 것 

**MVCC 장점** ✅
- **높은 동시성** 
- **데드락 감소** : MYSQL과 달리 Gap/Next-Key Lock을 사용하지 않아 대량 UPDATE 환경에서도 경합이 상대적으로 적음 


#### 다른 DB와 다른 전략 

- **PosgresSQL** : 변경 후 충돌 나면 롤백 
	- 동시성이 매우 뛰어남

- **MySQL** : 잠금을 먼저 걸어 변경 자체를 차단 
	- 최신 MySQL은 MVCC + Next-Key Lock 방식으로 팬텀 리드를 방지하기도 한다.
	- 하지만 이 방법은 락으로 잠궈서 물리적으로 팬텀 리드를 방지하는 것 
	- 장점 : 즉시성, 단순성 높음 
	- 단점 : 락을 사용하기 때문에, 경합이 발생하면 성능이 급격히 저하될 수 있다.


>[!tip] 낮은 격리 수준으로 Phantom Read를 방지한다는게 진짜 대단한거다.


## Serializable vs Repeatable-Read


### 예시 sample

| id  | value |
| --- | ----- |
| 1   | A     |
| 2   | A     |
| 3   | B     |
| 4   | B     |
위의 같은 테이블이 있다고 가정할 때, 
Transaction 1 : value가 A이면 B로 변경
Transaction 2 : value가 B이면 A로 변경
할거라고 가정하자.

```SQL
SELECT * FROM test
INSERT INTO test(id, value) VALUES (1, 'A')
INSERT INTO test(id, value) VALUES (2, 'A');
INSERT INTO test(id, value) VALUES (3, 'B');
INSERT INTO test(id, value) VALUES (4, 'B');
```

#### 예시 in Repeatable Read(or Read Commited)

```SQL
[트랜잭션 1]
BEGIN transaction isolation level Repeatable Read;

SELECT * FROM test;

UPDATE test SET value = 'B' WHERE value = 'A';

SELECT * FROM test;

[트랜잭션 2]
BEGIN transaction isolation level Repeatable Read;

SELECT * FROM test;

UPDATE test SET value = 'A' WHERE
```


```SQL 
✅트랜잭션 1 - UPDATE 후 COMMIT 전 READ
 id | value 
----+-------
  3 | B
  4 | B
  1 | B
  2 | B


✅트랜잭션 2 - UPDATE 후 COMMIT 전 READ 
 id | value 
----+-------
  1 | A
  2 | A
  3 | A
  4 | A

✅ 둘다 커밋 후 결과 
 id | value 
----+-------
  1 | B
  2 | B
  3 | A
  4 | A
```
- T1은 A → B로 변경
	- `id=1,2`는 T1의 영향 → `B`
- T2는 B → A로 변경
	- `id=3,4`는 T2의 영향 → `A`

>[!tip] PostgresSQL은 Snapshot 기반 
>- 각 트랜잭션은 시작 시점의 데이터를 기준으로 고정된 Snapshot을 사용
>- 따라서 트랜잭션1이 `COMMIT`을 해도 트랜잭션 2는 변경사항을 보지 못한다.
>- 위의 예시는, 서로의 변경 사항을 보지 못한 채 UPDATE를 실행시키는 것 
>- 효과 : Phantom Read 방지 + Repeatable Read 

>[!danger] 한계 - 논리적 일관성 문제 
>- 동일 조건에 대한 업데이트라도 트랜잭션 간 격리로 인해 충돌 없이 실행 ➡ 순차적 실행을 원할 시 원하는 결과가 나오지 않음 
>- 이런 논리적 일관성 문제를 해결하려면 경우 격리 수준을 `SERIALIZABLE`로 올려야 한다.


#### SERIALIZABLE - 논리적 일관성 해결 ✅
**작동 방식**
	- `SERIALIZABLE` 트랜잭션은 다른 현재 실행 중인 트랜잭션의 **변경 사항**(읽기/쓰기 종속성)**으로부터 완전히 격리**된다. 
	- 데이터베이스는 이러한 **종속성을 감지**하고, **충돌이 발생하면 한쪽 트랜잭션을 강제로 실패(롤백)시킨다.**

격리수준을 isolation으로 바꾸고 앞의 예시를 다시 실행한다고 가정하자
Transaction 1 `COMMIT` ➡ Transaction 2 `COMMIT`  ❌ 오류 발생 ❌
```TERMINAL
`ERROR: could not serialize access due to read/write dependencies among transactions` `DETAIL: Reason code: Canceled on identification as a pivot during commit attempt. Hint: The transaction might succeed if retried.`
```
PostgreSQL이 T1과 T2 간의 **읽기/쓰기 종속성**을 감지했기 때문
T1이 `A`를 `B`로 바꾸는 동안, T2는 `B`를 `A`로 바꾸려고 한 것을 감지 
> . `SERIALIZABLE` 격리 수준은 이러한 동시성 작업이 순차적으로 실행된 것처럼 보이게 해야 하므로, 데이터베이스는 T2의 커밋을 허용하지 않고 롤백시킨다.



>[!tip] 이러한 문제 해결은 SEREALIZABLE 말고 비관적 락으로도 가능하긴하다



### 틀렸던 문제

#### 문제 
Jeff executed the following update statement in the terminal on a table with 100 million rows;
```SQL
UPDATE TABLE STUDENTS SET GRADE = 100;
```

He realized that he forgot to add a where clause to only update the row where student id 50. Jeff executed a rollback immediately in the terminal.
```SQL
ROLLBACK;
```
He then rerun the statement again
```SQL
UPDATE TABLE STUDENTS SET GRADE = 100 WHERE STUDENT_ID = 50;
```
❓What is the state of the table after the last statement?

#### 해설 
처음에는 롤백했으니 이전에 선언한 UPDATE는 취소되는지 알았다.
하지만 아니였다 ➡ **자동 커밋 여부**를 생각해야함 
(그리고 BEGIN transaction도 없는데 왜 그런 생각을 했는지.....)

대부분의 CLI(MySQL, Postgres psql 등)에서는 각 SQL문이 끝나면 **자동으로 커밋**이 일어난다.
**첫 UPDATE를 시작하고 그다음 ROLLBACK을 해도 첫 UPDATE는 커밋된 상태로 ROLLBACK이 효과가 전혀 없다.**

