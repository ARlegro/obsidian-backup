> 배타적 Lock vs 공유 Lock 

### 1. Exclusive Lock 
특정 데이터(컬럼 또는 행)에 대해 **독점적인 권한**을 얻는 것을 의미
- 내가 어떤 데이터를 읽거나 업데이트하려 할 때, **다른 누구도 이 값을 읽거나 수정하지 못하도록** 하는 것
- 보안이나 일관성 유지를 위해 해당 데이터에 대한 **배타적인 접근 권한**을 확보
- 나만이 해당 데이터를 업데이트할 수 있도록 보장하며, 다른 트랜잭션이 몰래 데이터를 읽어가는 것을 방지

>[!tip] Exclusive Lock 획득 후, 트랜잭션이 해당 데이터를 **읽거나 수정하려고 시도하면 오류가 발생하거나 대기**

### 2. Shared Lock 
내가 어떤 데이터를 읽으려 할 때, **다른 누구도 이 값을 변경하지 못하도록** 하는 것
**여러 사용자(연결)**가 **동시에** 하나의 데이터에 대해 `Shared Lock`을 획득할 수 있다.

>여러 트랜잭션이 동시에 같은 데이터를 읽을 수 있지만, 아무도 해당 데이터를 변경할 수는 없다.


### 상호 배제 규칙 
- `Exclusive Lock`을 획득하려면, 해당 데이터에 **어떤 `Shared Lock`도 걸려 있지 않아야 합니다.**
- `Shared Lock`을 획득하려면, 해당 데이터에 **어떤 `Exclusive Lock`도 걸려 있지 않아야 합니다.**


### Lock의 장단점 

#### 장점 ✅
- **일관성 보장**: `Shared Lock`과 `Exclusive Lock`은 시스템의 **일관성**을 보장하는 데 매우 유용
- **읽기 일관성**: `Shared Lock`을 통해 특정 값을 읽는 동안 그 값이 변하지 않도록 
- **쓰기 일관성**: `Exclusive Lock`을 통해 내가 데이터를 수정하는 동안 다른 누구도 해당 데이터를 읽거나 수정하지 못하도록 하여, 다음에 읽는 사람이 항상 최신 값을 볼 수 있도록

#### 단점 💢

- **동시성 저하**: 잠금을 사용하면 **동시성(Concurrency)**이 저하될 수밖에 없다.
    - Lock은 다른 트랜잭션의 작업을 차단하여 지연이나 실패를 유발할 수 있다.
    - 은행 시스템의 경우, 밤중에 보고서 작업(대규모 `Shared Lock`)이 진행될 때 고객의 거래가 일시적으로 중단될 수 있다. (간혹 밤에 특정 거래가 막히는 이유가 여기에 있을 수 있다)
- **실패 가능성 증가**: 잠금 경합이 심하면 트랜잭션 실패(deadlock 또는 timeout) 가능성이 높다.


> 일관성 획득 But 동시성 단점 


