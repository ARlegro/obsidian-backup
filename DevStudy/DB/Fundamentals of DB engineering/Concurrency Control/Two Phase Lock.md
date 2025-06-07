https://code-run.tistory.com/38



### What is '2 phase lock'
데이터베이스 시스템에서 동시성 제어를 위한 가장 기본적인 비관적(pessimistic) 방법 중 하나
2phase에 걸쳐서 DB 락을 얻고 release한다???

### 단계 (간단히 - 뒤에 자세히)

#### 1. Growing 
>Lock 요청할 수 있는 단계 
- 각각의 트랜잭션은 트랜잭션 매니저에게 특정 데이터에 접근하기 위한 잠금을 요청
- 트랜잭션 매니저는 상황에 맞게 잠금 요청을 승인하거나 거부할 수 있다.

#### 2. Shrinking 
>Lock 해제할 수 있는 단계

 

![[Pasted image 20250605134951.png]]

### 2PL 이 필요한 예시 
- 영화관 자리를 예약한다고 가정하자.
- 자리는 절대 겹치면 안된다.
- 하지만 동시에 2명의 유저가 같은 자리를 예약하려하면 어떻게 되는지 SQL로 확인해보자 
```sql
 id | isbooked | name 
----+----------+------
 13 |          | 
(1 row)

-- T1 실행
postgres=# UPDATE seats SET isbooked=true, name='YH' WHERE id = 13;
UPDATE 1

-- T2 실행
UPDATE seats SET isbooked=true, name='KKK' WHERE id = 13;
대기 발생 

-- T1 COMMIT 
COMMIT;

-- T1이 COMMIT하자 T2의 대기가 풀림
-- T2 이제 COMMIT 
T2 COMMIT;

SELECT * FROM seats;
 id | isbooked | name 
----+----------+------
 13 | t        | KKK
(1 row)
```
> [!missing] 두 트랜잭션 모두 성공했지만 값은 하나다.



T2는 update 요청을 대기 중이다가 T1이 `COMMIT` 하면 T2도 UPDATE됐다는 알림이 뜬다.
여기서 T2가 `COMMIT`하면 성공적으로 된다. 이로 인해, 자리는 더블 예약이 된 것💢
>이러한 문제를 2PL이 해결해 줄 수 있다.
>(참고로 postgreSQL은 이론과 달리 UPDATE문 발생시 암묵적 배타락 걸어줌. 지금은 공부용이니 단순 UPDATE로는 락 안걸린다고가정)


### 2PL: Two-Phase Locking

>[!tip] UPDATE하려는 row의 Exclusive Lock을 얻을 것이다.
>- 트랜잭션에서 **정합성 있는 UPDATE**를 보장하기 위해,
>- 대상 Row에 대해 **배타적 잠금(Exclusive Lock)**을 걸고, 다른 트랜잭션의 간섭을 막는 구조

#### 핵심 키워드 : **`FOR UPDATE`⭐**
- `FOR UPDATE`는 **지정된 row에 대해 배타락**을 건다. 
- 즉, 다른 트랜잭션이 해당 row를 읽거나 수정하려 해도 **대기 상태에 빠짐**.

#### **✅Phase 1** - Lock 획득 단계 (Growing Phase)
```SQL 
SELECT * FROM seats WHERE id = 14 **FOR UPDATE** <<<< 이게 중요 
```
- 트랜잭션 T1은 **id가 14인 좌석에 대한 배타락을 획득**
- 이 상태에서는 **T1만 해당 row에 접근 가능**
- 아직 **락을 해제할 수 없음** → 해제는 Phase 2에서만 가능

>[!QUESTION] ❓이 상태에서 다른 트랜잭션이 접근하면 어떻게 될까❓
```SQL 
-- T2가 배타락 걸린 ROW에 접근하여 배타락을 얻으려 시도 시 
SELECT * FROM seats WHERE id = 14 FOR UPDATE
멈춘다.
```
- ❌ **즉시 실행되지 않고 대기**
- 이유: 이미 T1이 해당 row에 대해 배타락을 보유하고 있음
- PostgreSQL은 T1이 `COMMIT` 또는 `ROLLBACK`할 때까지 T2를 **대기 상태로 둠**


#### **✅Phase 2** - Lock 해제 단계 (Shrinking Phase)
```SQL
-- 배타락 걸고 업데이트 후 커밋 
UPDATE seats SET is_reserved = true WHERE id = 14;
COMMIT;
```
T1이 작업을 마치고 `COMMIT`하면,
- 락이 **해제**
- 대기 중이던 T2는 **락을 획득하고 실행 계속**

Note : `ROLLBACK`도 락을 해제함 


### 주의 : 추가 락 획득 불가 ❌

#### 이론 상 
2PL에서 Phase2에서 Lock을 해제한다.
근데, 한 번 해제하면 그 이후 추가로 Lock 획득이 불가능하다.
- **Growing Phase**: 락을 자유롭게 획득할 수 있지만, **해제는 금지**
- **Shrinking Phase**: 한번이라도 락을 해제하면, **추가 락 획득 금지**

>[!QUESTION] 왜 막아 놨을까? ➡ 직렬화 순서가 꼬이기 때문
>- 락을 해제한 시점 이후엔, 그 트랜잭션은 **자기보다 늦게 시작한 트랜잭션보다 먼저 끝났다고 간주됨**
>- 그런데 다시 락을 얻으면, **늦게 시작한 트랜잭션보다 더 늦게까지 작업하는 셈**
>- 결국 **타임라인이 모순**되고, **직렬 가능성이 깨지는 결과**가 생김



#### 실제론? ex. PosgreSQL 

PostgreSQL은 **2PL을 이론 그대로 적용하지 않는다.**  
실제로는 내부적으로 더 유연한 방식(MVCC + 다양한 락 모드)을 사용해서 필요할 때 락을 걸 수 있다.





Note : 최신 데이터베이스 시스템(PostgreSQL, MySQL InnoDB 등)은 2PL과 같은 락 기반 동시성 제어 방식 외에도 **MVCC**와 같은 다른 고급 동시성 제어 기법을 사용하여 읽기(Read) 작업의 동시성을 높이고 락 경합을 줄이기도 함 

하지만 쓰기(Write) 작업의 일관성을 위해서는 여전히 락이 중요하며, 그 기반에는 2PL(특히 Strict 2PL) 개념이 깔려 있다.


## 스프링 + JPA 에서 배타락 구현 

```java 
@Lock(LockModeType.PESSIMISTIC_WRITE)  
@Query("SELECT b FROM Board b WHERE b.id = :id")  
Optional<Board> findByIdForUpdate(@Param("id") Long id);
```

`@Lock(PESSIMISTIC_WRITE)`
- SELECT 시점에 DB row에 배타적 락 (`FOR UPDATE`) 걸기


>[!todo] 다른 Lock 모드
>- PESSIMISTIC_REA : 공유 락 (다른 트랜잭션은 읽기만 가능)
>- PESSIMISTIC_WRITE : 배타 락 (`FOR UPDATE`)
>- OPTIMISTIC : 낙관적 락 (@Version 기반)


> [!WARNING] 배타락 구현 시 성능 저하 및 데드락 발생 조심 
> - `PESSIMISTIC_WRITE`는 실제 DB 락을 사용하므로, **성능 저하 및 데드락 발생 가능성** 존재
> - 반드시 필요한 상황에만 사용
> - 트랜잭션 타임아웃 설정 (`@Transactional(timeout = 5)`) 등을 함께 고려하면 좋음 




2PL 말고도 더블 예약 문제를 해결하는 여러 방법이 있는데, 아래의 링크는 다른 방법 설명 
[[Preventing 'Double_Books' Without 2PL]]
