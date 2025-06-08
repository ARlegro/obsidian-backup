

### Preview 
- 쓰이는 격리 수준 : `READ COMMITTED`
- 장점 ✅
	- 동시성이 높다
	- 데드락 기회가 준다.
- 단점 💢
	- 충돌 시 트랜잭션이 롤백되어 오버헤드가 일어날 수 있다.


### 동작 원리 
- **데이터베이스 락을 직접 사용하지 않음:** 데이터베이스의 행 레벨 락(비관적 락)을 직접 사용하지 않고, 애플리케이션 레벨에서 `version` 컬럼을 통해 충돌을 감지한다.
- **성능 이점:** 락으로 인한 대기(블록)가 없어 동시성이 높은 환경에서 성능상 이점이 있을 수 있다. (충돌이 잦으면 재시도로 인해 불리할 수도 있음)
- **재시도 로직 필수:** 충돌 발생 시 예외가 발생하므로, 애플리케이션 코드가 이를 처리하고 필요하면 재시도하는 로직을 구현해야 한다.
- **JPA `@Version` 지원:** Spring Data JPA나 Hibernate는 `@Version` 어노테이션을 통해 낙관적 락을 매우 쉽게 적용할 수 있도록 지원한다.

#### Flow

| T1                                                               | T2                                              | DB                                       |
| ---------------------------------------------------------------- | ----------------------------------------------- | ---------------------------------------- |
| 트랜잭션 시작                                                          | 트랜잭션 시작                                         | ID : 1<br>Status : Free<br>Version : 1   |
| Read Row ID : 1<br>(Row Version = 1)<br>(쉐어락 걸림 - But 읽자마자 락 반납) | Read Row ID : 1<br>(T1과 똑같음)                    | ID : 1<br>Status : Free<br>Version : 1   |
| SELEDT FOR UPDATE<br>(버전 검증 일어남 + 배타락)                           |                                                 | ID : 1<br>Status : Free<br>Version : 1   |
| UPDATE                                                           |                                                 |                                          |
| COMMIT                                                           | (트랜잭션 중)                                        | ID : 1<br>Status : Booked<br>Version : 2 |
|                                                                  | `SELECT FOR UPDATE`<br>**(버전 검증 일어남 + 배타락 시도)** | ID : 1<br>Status : Booked<br>Version : 2 |
|                                                                  | **ROLLBACK됨 Cuz 버전이 이상해서**                      | ID : 1<br>Status : Booked<br>Version : 2 |


### 언제 낙관적 락을 쓸까?
- 읽기 작업이 훨씬 더 많은 애플리케이션에 적합 
- 비관적만큼 데이터 무결성을 보장할 수 없지만 교착상태에 빠지지 않아 성능 좋다.


---


## JPA, 스프링에서의 낙관적 락 이해 
https://www.baeldung.com/jpa-optimistic-locking
### 시작 - @Version
> @Version 애노테이션을 알아야 한다.
> 낙관적 락을 활성화하는 데 필수적 
> 스프링은 `@Version` 어노테이션이 붙은 숫자 타입 속성을 사용하여 낙관적 락을 지원

**@Version ❓**
- @Version이 붙으면 `UPDATE` 쿼리에는 **데이터베이스에 저장된 버전이 실제로 변경되지 않았는지 확인하는 `WHERE` 절이 포함**된다.
- 만약 변경되었다면, `OptimisticLockingFailureException`이 발생
- 만약 변경되지 않았다면, `UPDATE`가 커밋되고 version 속성의 값이 증가한다
- 규칙
	- 각 엔티티는 하나의 버전 속성만 가질 수 있다.
	- 버전의 속성 타입은 : int, Integer, long, Long, short, Short, Timestamp 가능 
	- 여러 테이블에 매핑된 엔티티의 경우 **'주 테이블'에 배치**되어야 한다.
	- Version속성을 직접 업데이트하거나 증가시켜서는 안된다.
		- 오로지 영속성 공급자만이 해야 함
		- 그래야 데이터의 일관성 유지됨 

> [!WARNING] 낙관적 락을 위해 반드시 @Version을 써야하는 것은 아니다.
> 단지 스프링에서 선호하는 방법일 뿐이다.


> **optimistic locking is based on detecting changes on entities by checking their version attribut**



나중에 추가하기 

https://www.baeldung.com/jpa-optimistic-locking


---
### 자바 예시

일단 테이블은 아래와 같았는데, Version 필드를 추가할 것 

```java
@Version
private int version;
```