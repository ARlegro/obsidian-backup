MultiVersion Concurrency Control


### 도입 
기존 SQL에서는 한쪽이 Read하면 다른 한쪽은 Writer를 못해서 동시에 처리할 수 있는 처리량이 줄어들어 성능에 안 좋은 영향이 있다.
이러한 문제를 해결하기 위해 MVCC가 개발이 됐다.
물론 MVCC에서도 두 트랜잭션이 write-writer 하는 경우에 블락이 되지만, write-read인 경우에는 블락되지 않아 동시에 처리할 수 있다는 장점이 있다.
![[Pasted image 20250607093820.png]]
(출처 : 쉬운 코드)


### 동작 원리


| 시간  | Transaction 1                    | Transaction 2                                                                      | 데이터    |
| :-: | -------------------------------- | ---------------------------------------------------------------------------------- | ------ |
|  1  | READ - Ｘ<br>(X = 10)             | Write -  X = 50<br>(FOR UPDATE ➡ 배타락)<br>➡ DB에 쓰여지지 않고 **트랜잭션 2만 알 수 있는 공간에 쓰여진다** | X = 10 |
|  2  | READ - X<br>(여전히 10)             |                                                                                    | X = 10 |
|  3  |                                  | COMMIT<br>(배타락 반환)                                                                 | X= 50  |
|  4  | READ - X <br>(격리수준에 따라 X 값이 다르다) |                                                                                    | X = 50 |
- MVCC는 commit된 데이터만 읽는다 
- **4번 설명** 
	- 격리 수준에 따라 Transaction 1이 읽는 X값이 다르다.
	- `READ COMMITED`이면 X = 50
		- **READ 시간 기준**으로 COMMIT된 데이터 읽음 
	- `REPEATABLE READ`이면 X = 10
		- **Transaction 시작 시간 기준**으로 그 전에 COMMIT된 데이터를 읽는다.
	- `SEIALIZABLE` 
		- RDBMS마다 동작이 조금 다르다.
		- MYSQL : MVCC가 아닌 LOCK으로 동작한다(나중에 공부)
		- **PostgreSQL : SSI기법이 적용된 MVCC로 작동**한다
			- SSI = Serializable Snapshot Isolation

### MVCC 중간 정리
- 데이터를 읽을 때 **특정 시점을 기준**으로 가장 최근에 COMMIT된 데이터를 읽는다.
	- 격리수준에 따라 다르다.
- 데이터의 변화이력을 따로 관리한다
- `READ`와 `WRITE`는 서로를 Block하지 않는다.


물론, Version(변화 이력)을 여러개 관리해야 돼서 메모리를 많이 쓴다.
그래도 장점은 `READ-WRITE` 충돌이 안 일어나 동시성 성능이 매우 좋다.
따라서 요즘의 RDBMS는 **성능이 좋아서 MVCC를 사용**한다.



### PostgreSQL의 LOST UPDATE 해결 

#### 흐름 
![[Pasted image 20250607102513.png]]
위의 사진과 같은 흐름이 있다고 했을 때, 
트랜잭션 2가 Write하려했지만 트랜잭션 1이 Write하느라 락을 걸어서 트랜잭션 2는 대기상태에 빠짐 
트랜잭션 2가 기다리면서 WRITE한 작업은 LOST UPDATE를 일으켜서 예상과 다른 결과가 나온다.
하지만, PostgreSQL에서는 격리수준을 바꿔주면 사진과 같은 결과가 나오지 않는다.
(First-Update Win 덕분 unlike MYSQL)

#### 격리수준 격상으로 Lost Update 막기 in PostgreSQL
>[!tip] PostgreSQL은 `REPEATABLE READ`일 때, 같은 데이터에 **먼저 UPDATE한 트랜잭션이 COMMIT되면 나중 트랜잭션은 ROLLBACK된다**. ⭐⭐
>- 이러한 현상을 First-Update-Win이라고 부름 
>- 일단, 트랜잭션마다 서로 다른 isolation level을 가질 수 있다(그러나, DB가 알아서 격리수준을 올려주지는 않는다)
>- 따라서 연관있는 트랜잭션 모두 즉, 양쪽 다 REPEATABLE READ로 바꿔주면 LOST UPDATE가 안 일어난다. ⭐⭐


위의 해결방법은 단순히 충돌한 트랜잭션이 롤백함으로써 하나의 트랜잭션만 성공시키게 하는 것이였다.
```TEXT

[상황]
However, such a target row might have already been updated (or deleted or locked) by another concurrent transaction by the time it is found. In this case, the repeatable read transaction will wait for the first updating transaction to commit or roll back (if it is still in progress). 

[락걸은 트랜잭션이 ROLLBACK될 경우 or 성공할 경우]
If the first updater rolls back, then its effects are negated and the repeatable read transaction can proceed with updating the originally found row. But if the first updater commits (and actually updated or deleted the row, not just locked it) then the repeatable read transaction will be rolled back with the message

[에러 메시지]
`ERROR:  could not serialize access due to concurrent update`

[이유]
because a repeatable read transaction cannot modify or lock rows changed by other transactions after the repeatable read transaction began.

[롤백되면 해야하는 일과 충돌이 없는 결과]
When an application receives this error message, it should abort the current transaction and retry the whole transaction from the beginning. The second time through, the transaction will see the previously-committed change as part of its initial view of the database, so there is no logical conflict in using the new version of the row as the starting point for the new transaction's update.

[출처]
in PostgreSQL DOC
https://www.postgresql.org/docs/current/transaction-iso.html -> REPEATABLE READ 
```
- 똑같은 Row를 수정하는 트랜잭션 1, 2  (when `REPEATABLE READ` in PostgreSQL)
- 1이 Lock걸어서 수정하고 있는데 2가 수정 시도하면 대기상태에 빠짐
- 1이 수정 후 커밋 ➡ 2는 에러 뜨면서 `ROLLBACK`
- 이 상황에서 **애플리케이션이 해야 할 일**
	- 현재 트랜잭션을 **중단(abort)** 하고 **전체 트랜잭션을 처음부터 다시 시도(retry)** 해야 한다
	- 이로 인해 재시도 할 때 논리적 충돌은 발생하지 않는다.
	  
- **재시도 시 논리적 충돌이 없는 이유**:
	- 두 번째 트랜잭션이 재시도될 때, 이 트랜잭션은 **이전에 커밋된 변경 사항(즉, 첫 번째 트랜잭션이 성공적으로 수행한 업데이트)** 을 데이터베이스의 **초기 뷰(initial view)** 의 일부로 보게 된다. 
	- 따라서 **새로운 트랜잭션의 업데이트를 위한 시작점으로 행의 새로운 버전을 사용**하는 것
	- 따라서, 어떠한 논리적 충돌도 발생하지 않는다
	  

> 참고로, 위의 First-Update-Win 내부 동작은 PostgreSQL에서 동작하는 특이한 흐름이다.
> 막상 MYSQL은 같은 격리수준이여도 저렇게 동작하지 않는다.


> 대안 : 격리수준 조정이 아니더라도 낙관/비관적 락으로도 해결 가능
> - `READ COMMITTED` + 비관적 잠금 → 대기시간 길어질 수 있음.
> - `REPEATABLE READ` + 재시도 → CPU 는 조금 더 쓰지만 락 대기시간이 짧아짐.


### 추가 : 격리수준을 높이지 않아도 충돌이 안되는 이유 (PostgreSQL)

#### 상황
```SQL 
SELECT * FROM test;
 price 
-------
    99
(1 row)
```
- 컬럼 제약조건 : 100이하 
- 트랜잭션 1 ➡ 가격 1증가 시도 (PRICE = PRICE + 1)
- 트랜잭션 2 ➡ 가격 1증가 시도 (PRICE = PRICE + 1)
![[Pasted image 20250607112226.png]]

#### 궁금증 
트랜잭션마다 각자 고유한 Version을 가지고 있기 때문에 양쪽 다 UPDATE하면 각 Version의 데이터를 99 -> 100으로 증가시켜 성공해야된다고 생각할 수 있다.
하지만 실제로는 뒤늦게 시작한 트랜잭션의 update가 제약조건 위반 메시지가 떴다.
이는 처음 트랜잭션의 UPDATE문을 반영한걸로 보인다. 왜❓
>[!QUESTION] 분명, 다른 버전으로 작업중일텐데 어떻게 `UPDATE`문에 반영됐을까❓

#### 해답 



**최신으로 커밋된 데이터를 기반으로 자신의 쿼리 조건을 다시 평가하고 연산을 수행**



















