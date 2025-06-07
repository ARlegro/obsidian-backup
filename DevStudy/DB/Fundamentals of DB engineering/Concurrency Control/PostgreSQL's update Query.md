
### 일반적 UPDATE
#### 설명명
어떤 데이터를 수정한다고 가정하자.
JPA의 경우 일단 그 데이터를 조회하고 JPA가 변경감지를 하면 UPDATE 쿼리를 날려준다.
아래는, Update 메서드와, 락관련 설정하나도 없이 JPA가 날려준 UPDATE 시 쿼리이다.
```java 
@Transactional
public void update(Long id) {  
    Board board = boardRepository.findById(id).orElseThrow(...);
    board.setTitle("update title");  
}
```

```SQL
Hibernate: 
    select
        b1_0.id,
        b1_0.content,
        b1_0.created_at,
        b1_0.title 
    from
        boards b1_0 
    where
        b1_0.id=?
Hibernate: 
    update
        boards 
    set
        content=?,
        created_at=?,
        title=? 
    where
        id=?
```

PostgreSQL은 UPDATE같은 변경 쿼리를 대상으로 암묵적 배타락을 걸어준다.
즉, 백엔드에서 UPDATE 시 명시적으로 락을 걸지 않아도 자동으로 걸어주는 것이다.

>[!tip] PostgreSQL의 암묵적 배타락 실험은 다음 링크 참고 : [[Preventing Double books without 2PL]]

#### 문제 ⛔
(읽기 현상에 대한 글은 이전에 많이 언급했던 내용이라 자세한 내용은 생략)
(참고 : [[Deep Dive Read Phenomena]], [[2. Isolation of ACID]])

**위의 방식의 문제 - 읽기 현상 문제** 
- 간단히 말하면, UPDATE를 하기 위해 조회를 했는데 이 때는 락이 걸리지 않는다. UPDATE 쿼리 시 PostgreSQL이 암묵적 행단위 배타락을 걸어주는 것이다.
- 근데 **문제**는 트랜잭션 내부에서 조회와 `UPDATE` 사이에 실행 대기시간이 있다면 **그동안 다른 트랜잭션이 해당 행을 또 UPDATE할 수** 있다.

>[!danger] 이렇게 된다면 *쓰기 왜곡 (ex. 무결성 제약 위배)* or *Lost Update* 문제를 야기 

> @Transactional이 있다고 해서 전 구간의 Lock이 보장되는 것은 아니다!!!


## 해결책 

### 비관적 Lock 방식 
UPDATE를 위한 조회 시점에 배타락을 거는 것이다.
배타락을 걸면 아무도 읽고, 쓰지 못하기 때문에 읽기현상 문제를 해결할 수 있다.
```JAVA
public interface BoardRepository extends JpaRepository<Board, Long> {
    // 조회 시점에 FOR UPDATE로 행 잠금을 건다.
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT b FROM Board b WHERE b.id = :id")
    Optional<Board> findByIdForUpdate(@Param("id") Long id);
}
```
위처럼 `LockModeType.PESSIMISTIC_WRITE`를 붙이면, JPA는 SQL 레벨로 `SELECT … FOR UPDATE`를 날려 조회 시점부터 **행 단위 배타락**을 즉시 취득
이렇게 하면 배타락 취득한 트랜잭션이 커밋 or 롤백할때까지 다른 트랜잭션은 해당행에 접근하지 못한다.
```SQL
Hibernate: 
    select
        b1_0.id,
        b1_0.content,
        b1_0.created_at,
        b1_0.title 
    from
        boards b1_0 
    where
        b1_0.id=? for no key update
Hibernate: 
    update
        boards 
    set
        content=?,
        created_at=?,
        title=? 
    where
        id=?
```


`FOR no key UPDATE`쿼리가 생겼다. 이것도 배타락이라고 보면 된다(자세한건 뒤에 참고)




#### 번외 : FOR UDPATE 가 아닌 FOR NO KEY UPDATE인 이유 

>[!tip] 동시성 제어 강도 조절 
>- 선 결론 : 외래 키 참조는 허용 + 행 자체의 Modify는 방지

배타락을 걸었기에 `FOR UPDATE`를 기대했다.
하지만 실제로는 `FOR no key UPDATE`쿼리가 생겼다.

❓What is `FOR NO KEY UPDATE`❓
- 이 쿼리는 PostgreSQL 9.3 이후 버전부터 제공하는 것으로
- **행(row) 자체의 상태 변경(UPDATE/DELETE)만 막고, 외래 키→참조 무결성(foriegn key constraint)을 확인하기 위한 조회만은 허용**한다.
- 즉, 완전한 FOR UPDATE 보다는 **충돌 범위를 좁힌 것**
https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS

🥊반면, 일반적인 `FOR UPDATE`는
- 너무 강력한 Lock이라, 
- 동일한 행을 참조하는 외래 키 검증 작업조차 막아버린다. 
- FK 참조까지 막아버리면 다른 테이블의 데이터 삽입도 막아버릴 수 있음 💢

최신 PostgreSQL에만 해당하는 것(이전 버전 or 다른 DB는 `FOR UPDATE`쿼리 생성)


NO KEY UPDATE로 KEY는 참조하도록 허용 
해당 키를 참조하며 다른 테이블 데이터 생성
근데 해당 로우를 다루는 트랜잭션에서 DELETE
결국 없는 행을 참조하고 데이터 생성 
예상 결과 : PK인 데이터 삭제 + FK가진 데이터 생성 ➡ 참조 무결성 위배 

## 참조키 제약 없이 테스트 

```sql 
CREATE TABLE comments
(
    id SERIAL PRIMARY KEY,
    board_id INTEGER REFERENCES boards(id),
    comment TEXT
)
```

참고 : 기본 deleteById 사용 시 JPA 쿼리 - for update같은 것 없다.
```SQL
Hibernate: 
    select
        b1_0.id,
        b1_0.content,
        b1_0.created_at,
        b1_0.title 
    from
        boards b1_0 
    where
        b1_0.id=?
Hibernate: 
    delete 
    from
        boards 
    where
        id=?
```



### FOR UPDATE 시 삽입 + 참조키 기본 제약

트랜잭션 A = PK가진 테이블(Board) 다루는 트랜잭션
트랜잭션 B = FK가진 테이블(Comment) 다루는 트랜잭션

1. **FOR UPDATE 후 FK가진 데이터 생성**
		![[Pasted image 20250606112450.png]]
	- 트랜잭션 A : Boards테이블을 `FOR UPDATE`로 **배타락**을 걸은 상태.
	- 트랜잭션 B : 자식 테이블에 데이터를 삽입하는 명령어 실행 
	- 이때 **트랜잭션 B는 대기상태에 빠짐** (A가 커밋 or 롤백할때까지)

2. **배타락 걸은 ROW DELETE**
		![[Pasted image 20250606113049.png]]
	- 트랜잭션 A : 1번에서 배타락을 걸은 행을 `DELETE` 했다.
	- 트랜잭션 B : 아직도 대기 상태 

3. **A COMMIT**  (2번 사진 참고)
	- B는 대기상태에 풀렸지만, 외래 키가 없어 오류가 떴다.

> 현재, 이 상황은 `FOR UPDATE`라는 아주 강한 락으로 인해 쓰기 왜곡을 방지해줬다.


### FOR NO KEY UPDATE 시 삽입 + 참조키 기본 제약 
A, B 는 이전 예시와 같다.

#### 시나리오 별 설명 
1. **FOR NO KEY UPDATE 후 FK 테이블 삽입 시도** 
		![[Pasted image 20250606113836.png]]
	- 트랜잭션 A : `FOR NO KEY UPDATE`로 락을 걸음 
	- 트랜잭션 B : 자식 테이블에 `INSERT` 시도 (성공)
	- 이번에는 B가 대기상태에 빠지지 않았다 Cuz `FOR UPDATE` 에서 `NO KEY`가 추가되어 KEY 참조에 락을 걸지 않았기 때문이다.
	- **참고로 `INSERT`시 부모행은 공유락 걸림** 
		- PostgreSQL은 자식행을 `INSERT`할 때부모행을 KEY SHARE (공유락) 잠금  
		- `NO KEY UPDATE` 모드는 `FOR KEY SHARE` 잠금과 충돌하지 않으므로(잠금을 동시에 허용하므로) 이런일이 가능했던 것 
		- 즉, 현재 걸려 있는 Lock은 총 2개인 것 
	  
2. **PK 테이블 값 데이터 삭제 시도** 
		![[Pasted image 20250606113937.png]]
	- 이번에는 PK가진 테이블의 트랜잭션이 대기상태에 빠졌다....
	- **이유 : 1번에서 B가 부모 테이블에 공유락 걸어서**
		- A트랜잭션이 DELETE를 시도하려 하는데 대기에 빠졌다.
		- 이유 : `DELETE`는 내부적으로 `FOR UPDATE` or `FOR NO KEY UPDATE`가 필요하다. 하지만 이는 `FOR SHARE`와 충돌하기 때문에 기다려야 하는 것이다. (참고 : [[Row-level locks - PosgreSQL]])
		  
3. **트랜잭션 B `COMMIT`**
		![[Pasted image 20250606114332.png]]
	- 트랜잭션 B : COMMIT ➡ 성공
	- 트랜잭션 A : 대기중이던 `DELETE`문이 오류가 뜸 
	- `DELETE`문을 대기중이던 A는 외래키 제약조건을 위배했다고 경고가 날라왔다.

 
> [!INFO] `ON DLETE` 제약조건
> -  `NO ACTION`(기본값 - 현재) : DELETE 문이 실행될 때 자식 행이 있다면 실패 
> - `RESTRICT` : DELETE 문이 실행되기 전에 자식 행이 있다면 즉시 실패 
		  
#### 중요 매커니즘 이해 
>[!QUESTION] 트랜잭션 A가 읽고 있는 스냅샷의 데이터는 외래키 제약조건을 위배하지 않았는데 어떻게 이런 일이 발생했을까❓

이를 이해하기 위해서는 **1) PostgreSQL의 격리 수준 특징**과 **2) Refresh 내부 메커니즘** **3) DELETE의 체크 트리거**를 을 알 필요가 있다.

**✅필요한 사전지식** 
1. **PostgreSQL의 격리 수준 = `Read Committed`**
	- `COMMIT`된 것을 볼 수 있다.
	- 이 격리수준에서는, 트랜잭션이 **`SELECT`를 실행할 때마다 그 순간의 스냅샷**을 새로 잡는다.
2. **PostgreSQL은 COMMIT이 일어나면 적용된 행(ROW)의 값을 Refresh하는 매커니즘**이 있다.
3. `DELETE`명령은 실행 전 `SELECT` 문을 통해 자식 행이 있는지 확인한다.

**위의 예시 돌아보기**
- `DELETE`를 시도하는 A는 공유락에 걸려서 대기 상태 
- **B 커밋** (**이후가 중요⭐**)
- 이후 A 실패 

**✔A가 대기 ~ 실패하기 전까지 돌아보기** 
- B의 변경사항이 적용된 행은 Refresh되고 A는 Lock을 얻게 된다.(이때 락 수준은 `FOR UPDATE`)
- 이제 A의 `DELETE` 문이 실행되는데 기본 옵션이 `NO ACTION`이다.
- `DELETE` 직전에 실행되는 **FK 체크 트리거**가 있는데, 이는 `SELECT` 문을 사용해서 **자식 행이 있는지 확인**한다. 
- 이 때 격리수준은 `READ COMMITED`이기 때문에,  확인하는 **`SELECT`는 최신 스냅샷**이다.

>[!tip] 즉, PostgreSQL은 **락이 해제된 후에도 쿼리의 조건을 다시 평가**하여 최종적인 결정(업데이트 여부)을 한다.

> [!TIP] 핵심 : PostgreSQL은 락에 의해 대기했던 트랜잭션이 재개될 때, **변경된 데이터를 읽어와 자신의 쿼리 조건을 다시 확인**하는 과정을 거쳐 일관성을 유지



> [!INFO] 위의 내용 회고 
> - 애초에 이 실험이 이 동작을 테스트하려한게 아니라 조금 당황했다.(이거에 빠져서 파보니 이전에 이 실험을 한 이유를 까먹었다....)
> - 아무튼, 결과가 이해가 되지 않았는데 이전에 공부했던 PostgreSQL의 내부 동작흐름을 떠올려보니 이해가 됐다.
> - 참고 : [[Preventing Double books without 2PL]] << 여기서 비슷한 일이 벌어졌었다.

자세한 PostgreSQL 분석 : [[Deep dive Postgre's Isolatition Behavior]] ⭐⭐⭐


[[Preventing Double books without 2PL]]
https://www.postgresql.org/docs/17/transaction-iso.html -> 13.2.1. Read Committed Isolation Level