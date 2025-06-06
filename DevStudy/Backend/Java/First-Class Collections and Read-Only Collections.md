---
dg-publish: true
dg-home: false
---
> 일급 컬렉션과 읽기 전용 컬렉션

객체 지향 생활 체조 원칙 중 ‘일급 컬렉션 사용’ 원칙이 있다.

이 글은 이 원칙을 적용하면서 공부한 내용이다.

## **관심 계기 : 왜 쓰게 됐는가???**

프로젝트를 하면서 spring-batch 작업을 담당하고 있었다.<br>
멀티스레드 환경에서 4개의 Thread가 각각 외부 API를 호출하는 Flow를 짜고 각각 가져온 데이터를 배치의 ExecutionContext에 담아서 공유하는 과정에서 단순히 컬렉션을 담으면 위험할 것 같다는 생각이 들었다.
<br>
또한, 해당 컬렉션을 이용한 관련 비즈니 로직이 겹치기 때문에 좀 더 객체 지향스럽게 코드를 변경하고 싶었다.
<br>
참고 : 아래의 사진은 내가 짜고 있는 코드의 전반적인 흐름이다.
![[Pasted image 20250517211111.png]]



각 스레드가 외부 ApiStep이 ExecutionContext의 자료를 이용하는 상황

## 일급 컬렉션(First-Class Collection)

### ✅ 개념

- Collection(ex. List, Set, Map 등)을 Wrapping하면서 그 Collection 외 다른 필드가 존재하지 않는 구조를 의미한다.

```java
public class Orders {
    private final List<Order> orders;

    public Orders(List<Order> orders) {
        // this.orders = new ArrayList<>(orders); // 방어적 복사 예시인데 뒤에서 
        this.orders = orders;
    }

    public void add(Order order) {
        orders.add(order);
    }

    public List<Order> getOrders() {
        return Collections.unmodifiableList(orders); // 불변 보장
    }
}
```

### **✅ 효과**

컬렉션을 굳이 왜 Wrapping할까???

1️⃣ SRP 준수

- 해당 컬렉션 자체에 대한 책임과 도메인 로직을 한 객체에 몽아 관리할 수 있다.
- 예를 들어, Orders들의 가격의 합을 구하는 로직을 만든다고 할 떄, Orders를 사용하는 곳에서 비즈니스 로직을 만드는 것이 아니라 일급 컬렉션에 따로 비즈니스 메서드를 만들어 캡슐화 및 SRP 원칙을 강활 수 있다.

2️⃣ 불변성 유지

간혹 collection타입에 final을 걸어 놓으면 내부값도 불변한다고 오해하는 사람이 있지만 그건 잘못됐다.

아래의 코드를 예시로 보자

```java
@Getter 
public class Orders {

  private final List<Order> orders;

//******************************
  List<Order> orders = orders.getOrders();
  orders.clear();
}
```

- 과연 위의 코드가 clear가 될까?? Yes
- orders 클래스의 orders 변수를 꺼내서 외부에서 clear를 할 수 있다.

  > 즉, 단순히 final로 선언되더라도 내부 요소는 바뀔 수 있다는 것이다.

- 만약, 이 orders로 비즈니스 로직만 해야하는데, 실수로 올바르지 않은 코드를 작성하면 이는 큰 사고로 이어질 수 있다.


- ✅ 간단한 해결

  이를 해결하는 여러 방법들 중 가장 간단한 방법은 Getter를 없애고 외부에서 메서드로만 접근하도록 하는 방법이 있다. 아래의 예시를 보자

    ```java
    public class Orders {
      private final List<Order> orders;
      
      public Orders(List<Order> orders){
    		 this.orders = orders;
      }
    ```

  위의 코드에서 외부에서 orders를 접근하려면 어떻게 해야할까???

  못한다. 이렇게 캡슐화를 시켜놓음으로써 외부에서 함부로 Collection을 변경하는 것을 막아 안정성을 확보할 수 있다. 만약 추가, 제거 로직이 필요하면 Orders 클래스에 따로 메서드를 만들면 됨 (이건 3번 내용)


<aside>
💡

**번외 : 추가 해결법**

이러한 해결방법 말고 뒤에서 배울 unmodifiableCollection 혹은 방어적 복사를 할 수 있는데 자세한건 나중에

</aside>

**3️⃣ 상태와 행위를 한 곳에서 관리**

엄청난 장점 중 하나이다. (1번과 얼추 비슷한 내용이다)

일급 컬렉션을 만들면 데이터와 로직이 동시에 존재하는 클래스를 만들 수 있다.

아래의 코드처럼 컬렉션의 값과 그 컬렉션을 이용해서 비즈니스 로직을 짜는 클래스를 말하는 것이다.

```java
public class Orders {
  private final List<Order> orders;

  public Orders(List<Order> orders) {
    this.orders = orders;
  }

	// 외부에서 List<Order>에 order를 추가하고 싶을 시   
  public void addOrder(Order order) {
    orders.add(order);
  }
  
	// 외부에서 모든 Order의 가격을 합하는 비즈니스 로직 수행 
  public int getTotalPrice() {
    int sum = 0;
    for (Order order : orders) {
      sum += order.getPrice();
    }
    return sum;
  }

```

아래는 내가 프로젝트를 하면서 만들고 있는 일급 컬렉션

```java
@Getter
public class Keywords {

  private final List<String> keywords;

  public Keywords(List<String> keywords) {
    this.keywords = Collections.unmodifiableList(keywords);
  }

  // O(N2)인데... 나중에 Hash 구조로 다 바꿔야할 것 같네
  public List<ArticleApiDto> filter(List<ArticleApiDto> articleApiDtos) {
    return articleApiDtos.stream()
        .filter(this::isContainsIn)
        .toList();
  }

  public boolean isContainsIn(ArticleApiDto articleApiDto) {
    String summary = articleApiDto.summary();
    //summary.toLowerCase();
    return keywords.stream()
        .anyMatch(summary::contains);
  }
```

## 일급 컬렉션 안전하게 사용하기

### 1. 멀티스레드 환경 - 방어적 복사, 내부에서 구현

멀티스레드 환경에서 생기는 문제가 있다.

아래의 코드를 보고 멀티스레드 환경에서 어떤 문제가 있을까??

```java
**[멀티스레드일 경우]**
public class Orders {

  private final List<Order> orders;

  public Orders(List<Order> orders) {
    validateOrderPrice();
    this.orders = orders;
  }

  // 검증로직 
  public void validateOrderPrice() {

  }
}
```

✅ **상황**

- 외부에서 List<Order>를 만들고 생성자에 넣기
- 컬렉션 내부의 속성들을 검증
- 검증 완료 시 넣기

**💢 멀티 스레드에서는 문제 발생**

- validateOrderPrice하는 와중에 외부에서 orders값을 변경시킨다면??
- 검증이 제대로 안된 값이 들어가 버릴 수 있다.

**☑️ 해결방법 1. 방어적 복사**

```java
**[멀티스레드일 경우]**
public class Orders {
  private final List<Order> orders;

  public Orders(List<Order> orders) {
	  List<Order> copyOrders = new ArrayList<>(orders);
	  validateOrderPrice();
    this.orders = orders;
  }
```

위의 코드에서 외부에서 주입한 orders와 copyOrders는 다른 참조값을 가지게 된다.

이렇게 되면, 검증 로직이 진행될 떄, 외부에서 주입한 orders가 변경되더라도 전혀 문제가 발생하지 않는다.

**☑️ 해결방법 2. Order 생성로직을 내부에서**

```java
public class Orders {
  private final List<Order> orders;

  public Orders(List<String> names) {
	  List<String> copyNames = new ArrayList<>(names);
	  validateDuplicatedName();
	  
    this.orders = copyNames.stream()
                .map(carName -> new Car(carName))
                .toList;
  }
```

### 2. 읽기 전용 컬렉션으로 변경

자바에서는 Collection을 변경할 수 없게 만드는 유틸이 있다.

바로 `Collections.unmodifiable~~~`이다.

이것을 사용하면 이 컬렉션은 읽기만 가능한 읽기 전용 컬렉션이 되어, 객체의 참조값을 알아도 내부를 전혀 바뀔 수 없다.

```java
public class Keywords {

  private final List<String> keywords;

  public Keywords(List<String> keywords) {
    this.keywords = Collections.unmodifiableList(keywords);
    //Collections.unmodifiableMap()
  }
```
