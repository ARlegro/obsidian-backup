
>[!SUCCESS]  복습 
>- 이전에 동시성 이슈였던 더블 예약을 막기 위해 2PL 배타락을 사용했다 
>- 링크 : [[Two Phase Lock]]
>- 이번에는 다른 방식으로 더블 예약 문제를 막아볼 것 


간단하게 생각하면 이전에 있던 더블 예약 문제에서 조건을 더 추가하면 해결되지 않을까? 라는 생각을 하기 쉽다. 아래처럼 
```SQL
UPDATE seats SET isbooked = 1, name='YH' WHERE id = 1 and isbooked=0;
```
- `WHERE` 절에 isbooked 조건을 추가했다

>결론부터 말하면 PostgreSQL기준으로는 이런 방법으로 방지가 가능하다
>하지만, 결과가 아닌 **내부 원리**를 알아야한다. 이건 DB마다 다르다.

그렇다면, 아래처럼 2개의 Transaction을 적용했을 때 무슨 일이 일어날지 확인해보자 
```SQL
 id | isbooked | name 
----+----------+------
  1 |   false  | 
(1 row)

-- T1 
BEGIN transaction

-- T2
BEGIN transactino 

-- T1 업데이트
UPDATE seats SET isbooked = true, name = 'YH' WHERE id = 1 and isbooked = false;

-- T2 업데이트
UPDATE seats SET isbooked = true, name = 'ABCD' WHERE id = 1 and isbooked = false;
❌ 대기 발생 

-- T1 COMMIT -> T2 대기 풀리고 T2 UPDATE 실패함 💢
COMMIT; 

❓WHY happens❓
```
`FOR UPDATE` 같이 명시적으로 배타락을 안 걸었음에도 락이 걸렸다.
그것은, `UPDATE`라서 DB가 알아서 락을 걸어준다.
>[!TIP]  `SELECT ... FOR UPDATE` 같은 명시적인 락을 사용하지 않아도, `UPDATE`, `DELETE`, `INSERT`는 내부적으로 **필요한 수준의 row-level lock을 자동으로 획득**  #Implicitly_Lock 
> - PostgreSQL에서 **자동으로 Exclusive Lock**을 건다.

⭐⭐ 배타락으로 인해서 T2는 Lock 대기가 풀려도 UPDATE가 실패하게 된다.

결과가 중요한게 아니라, 이게 왜 일어나는지, PostgreSQL은 어떻게 하는지 알아야 한다.

>[!QUESTION] 왜 이런 일이 일어났을까❓ ⭐⭐
>- **Transaction이 `COMMIT`되면 Isolation Level이 원래대로(Read Committed) 돌아온다**
>- **이 동작의 원인⭐⭐⭐**
>	1. PostgreSQL의 **`READ COMMITTED` 격리 수준**과,
>		- T2는 대기가 풀리기 전 Read Committed수준에서 Commit 데이터를 봐버리게 되는 것  (heap에서)
>	2. `COMMIT`된 값들을 읽을 수 있도록 행의 값을 **새로 고치는(refresh) 내부 메커니즘** 덕분에 가능 
>		- PostgreSQL은 락이 해제된 후에도 쿼리의 조건을 다시 평가하여 최종적인 결정(업데이트 여부)을 내린다.
>- 이로 인해, WHERE절은 False가 되어서 UPDATER가 이루어지지 않은 것이다.

자세한 PostgreSQL 분석 : [[Deep dive Postgre's Isolatition Behavior]]


> 이러한 내부 동작원리는 DB 시스템마다 다르다.



### 위 방식의 단점 
위의 방식(where절 빡빡하게)대로 해도 더블 예약을 방지했다.
But 몇가지 단점이 있다.
1. `READ COMMITTED` 모드에 있어야 함 
2. 단일 `UPDATE` 문에 종속적인 제어
	 - UPDATE 방식은 데이터베이스가 `UPDATE` 문을 처리하는 **내부적인 방식**에 전적으로 의존하게된다.
	 - 트랜잭션 내에서 **여러 단계를 거쳐야 하는 복잡한 비즈니스 로직**을 구현할 때는 **`SELECT ... FOR UPDATE`와 같은 명시적 락 방식이 훨씬 더 예측 가능하고 안전하며, 개발자에게 더 많은 제어권**을 제공한다



