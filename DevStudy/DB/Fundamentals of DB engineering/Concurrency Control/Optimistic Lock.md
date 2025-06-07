

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

#### @Version ❓
- 스프링 데이터(Spring Data)는 `@Version` 어노테이션이 붙은 숫자 타입 속성을 사용하여 낙관적 락을 지원
- @Version이 붙으면 `UPDATE` 쿼리에는 **데이터베이스에 저장된 버전이 실제로 변경되지 않았는지 확인하는 `WHERE` 절이 포함**된다.
- 만약 변경되었다면, `OptimisticLockingFailureException`이 발생


### 자바 예시

일단 테이블은 아래와 같았는데, Version 필드를 추가할 것 

```java
@Version
private int version;
```