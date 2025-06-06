---
dg-publish: true
dg-home: false
---

## 개념 및 특장점

**개념**

- Java 21에서 추가된 **가벼운 스레드 구현**
- 기존 `Thread`보다 메모리 소비가 적고, OS 스레드보다 많은 수를 생성 가능해서 주목받고 있음

```java
// 스프링이 제공하는 가상 스레드 실행기
VirtualThreadTaskExecutor springVte = new VirtualThreadTaskExecutor();
// 자바 JDK 21에서 제공하는 가상 스레드 실행기
ExecutorService jdkVte = Executors.newVirtualThreadPerTaskExecutor();

System.out.println("스프링 가상 스레드 : " + springVte.getClass());
//class org.springframework.core.task.VirtualThreadTaskExecutor
System.out.println("JDK 가상스레드 : " + jdkVte.getClass());
// class java.util.concurrent.ThreadPerTaskExecutor

vte.submit(Runnable or Callable);  <<< 둘 다 가능 
```

스프링이 제공하는 가상스레드 Executor가 있고, 자바 jdk에서 제공하는 가상 스레드 Executor가 있다.

둘의 차이는 나중에 알아보자

### **장점 정리**

1. **경량화된 스레드**
    - **기존의 플랫폼 스레드 : 운영체제(OS) 스레드와 1:1로 매핑**되기 때문에, 많은 수의 스레드를 생성하면 **컨텍스트 스위칭 비용이 증가하고 메모리 부담이 커짐**
    - **가상 스레드** : **JVM 내부에서 관리**되며, **OS 스레드와 직접 연결되지 않음** → 훨씬 가벼움.
    - 따라서, **수천~수백만 개의 동시 실행 작업을 처리** 가능

    <br>

1. **블로킹 I/O 작업에서 효율적**
    - 가상 스레드는 **블로킹 I/O(Network, DB, File)에서 큰 성능 향상**
    - **기존 플랫폼 스레드** : 블로킹 상태에서 **OS 자원을 차지하면서 기다림** → 비효율적
    - **가상 스레드** : **블로킹 발생 시 자동으로 OS 스레드에서 해제되며, 다른 가상 스레드가 실행됨**
        - 즉, **블로킹 시간 동안 OS 스레드가 낭비되지 않음**
        - 대량의 블로킹 I/O 작업이 있는 서버(예: HTTP 통신, DB 요청, 파일 처리 등)에서 성능 최적화
        - 이로 인해, **`CompletableFuture`나 `Reactor` 같은 복잡한 비동기 처리 없이도 동기 코드 스타일 유지 가능**
    - 더 적은 플랫폼 스레드로 **더 많은 요청 처리 가능**
   
   <br>

1. **기존 `Thread`보다 컨텍스트 스위칭 비용이 낮음**
    - **기존 플랫폼 스레드** : OS가 **하드웨어 레벨에서 컨텍스트 스위칭**을 수행 → 비용 큼
    - **가상 스레드** : **JVM이 직접 관리하는 사용자 모드 컨텍스트 스위칭을 수행** → 비용이 낮음. 다시 말해, 가상 스레드는 **JVM 내부 스케줄링을 사용하므로 전환 비용이 적음**
    - **컨텍스트 스위칭 낮은 이유 - 자세한 분석**

      ✅ 가상 스레드는 OS 스레드와 1:1 매핑되지 않고, **많은 가상 스레드가 소수의 플랫폼 스레드에서 실행됨.**

      ✅ **I/O 블로킹이 발생하면 자동으로 해제되고, 다른 가상 스레드가 실행됨.**

      ✅ **레지스터 및 캐시 오버헤드가 줄어들어 성능이 향상됨.**

    - CPU 사용률 증가하긴 하지만 CPU가 충분하다면 컨텍스트 스위칭 비용을 줄여주는 것이 커서 → 처리량(Throughput) 향상

        
        💡 참고 - 컨텍스트 스위칭 비용이란?

        - CPU가 현재 실행 중인 스레드의 상태(Context)를 저장하고, 다른 스레드의 상태를 복원하는 과정
        - CPU는 한 번에 하나의 스레드만 실행할 수 있으므로, **멀티스레드 환경에서 여러 작업을 실행하려면 스레드 간 전환(Context Switching)이 필요**


1. **높은 동시성(Concurrency) 처리 능력**
    - 적은 플랫폼 스레드로도 **수만 개의 동시 요청을 처리 가능**
    - `ForkJoinPool`이나 `CompletableFuture` 같은 복잡한 비동기 처리를 하지 않고도 **비동기적 동작 가능**
        - Future, CompletableFuture 이런 복잡한 비동기 API 없이 코드를 짤 수 있기에 디자인 적으로 동기식으로 가능
    - **ThreadLocal 사용 가능** (하지만 일부 제한 있음)
   
   <br>

1. **종료 관리 용이**
    - **JVM이 필요할 때만 실행하고, 사용이 끝나면 자동으로 정리됨**.
        - 작업이 제출될 때마다 새로운 가상 스레드를 생성 후 실행한 뒤 자동 종료
        - 다만, **장시간 실행되는 가상 스레드**가 있다면 `try-with-resources` 또는 `close()`를 활용해 명확하게 종료할 수도 있음.

            ```java
            try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
                executor.submit(() -> System.out.println("Virtual Thread Running"));
            }  // try 블록이 끝나면 자동으로 종료됨!
            ```

    - **자동 관리되는 스레드**이므로 `shutdown()`이나 `close()`를 명시적으로 호출할 필요가 없다.
    - OS는 일반적으로 스레드 풀을 관리하지만, JVM은 **가상 스레드를 필요할 때만 실행**하여 CPU 리소스를 최적화
    - 즉, `ExecutorService`가 **가상 스레드를 관리**하므로, **사용자가 직접 종료할 필요 X**

**가상 스레드의 주요 장점 요약**

| **장점** | **설명** |
| --- | --- |
| **1. 경량화된 스레드** | 기존 스레드보다 훨씬 적은 메모리 사용, 수천 개 이상 생성 가능 |
| **2. 블로킹 I/O 최적화** | 블로킹 시 OS 스레드를 점유하지 않아 성능 저하 없음 |
| **3. 컨텍스트 스위칭 비용 절감** | JVM이 관리하므로 OS 레벨 컨텍스트 스위칭보다 비용이 낮음 |
| **4. 높은 동시성 지원** | 적은 OS 스레드로도 수많은 요청 처리 가능 |
| **5. 종료 관리 용이** | 가상 스레드는 **자동 관리되는 스레드이다.** |

**요약**

기존 플랫폼 스레드(커널 스레드)보다 가볍고, 동시성(Concurrency) 높은 작업을 효율적으로 처리할 수 있도록 설계됨. 특히 블로킹 I/O 작업을 많이 처리하는 애플리케이션에서 강력한 성능

**단점**

**CPU 집약적 작업에는 적합하지 않음**

- 가상 스레드는 **블로킹 I/O 작업**에 최적화
- **이유 분석**
    - **가상 스레드는 OS 스레드와 1:1 매핑되지 않음** → 실행 도중 다른 가상 스레드로 교체될 가능성이 있음.
    - CPU 연산이 끝날 때까지 컨텍스트 스위칭이 적은 **고정된 플랫폼 스레드**가 더 효율적일 수 있음.


**가상 스레드는 기본적으로 병렬성이 아님**

- 가상 스레드는 **동시성을 높이는 방식**으로 동작함
- 즉, **코어 개수만큼 병렬 실행되지는 않고, 많은 요청을 처리할 수 있도록 최적화**

**JVM 오버헤드 증가 가능**

가상스레드 자세한건 나중에

## 생성 예시

### **V1 - 미리 만들어 놓기**

```java
private final ExecutorService ves = Executors.newVirtualThreadPerTaskExecutor();

public void executeTasks() {
    Future<?> future1 = ves.submit(() -> isExistUserForAsync(channelId));
    Future<?> future2 = ves.submit(() -> isExistChannelForAsync(channelId));

    try {
        future1.get(); // 작업이 끝날 때까지 기다림
        future2.get();
    } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
    }
```

- 이렇게 미리 생성해놓고 갖다 써도 된다

> But 별로다
>

**별로인 이유**

1. **가상 스레드는 풀(Pool)이 필요 없음**
    - `newVirtualThreadPerTaskExecutor()`는 **매 작업마다 새로운 가상 스레드를 생성**하므로, `ExecutorService`를 따로 유지할 필요가 없음.
    - 그냥 **필요할 때 `Thread.startVirtualThread()`를 호출하는 것이 더 직관적이고 성능상 유리**.
2. **불필요한 리소스 관리**
    - `newVirtualThreadPerTaskExecutor()`는 내부적으로 **ForkJoinPool**을 사용하지만, **스레드를 재사용하지 않음**.
    - `ExecutorService`를 미리 생성해 놓으면 **추가적인 메모리 점유만 하고 실질적인 장점이 없음**.
3. **코드가 더 복잡해짐**
    - 단순히 `Thread.startVirtualThread()`를 호출하면 끝날 일을, `ExecutorService.submit()` 형태로 감싸야 해서 코드가 복잡해짐.

### V2 - 그때 그때 만들기

> 이 방식이 효율적이지만, **편의성은 V1이 좋아서 V1 or V3 많이 씀**
>

```java
Thread vt1 = Thread.startVirtualThread(() -> isExistUserForAsync(channelId));
Thread vt2 = Thread.startVirtualThread(() -> isExistChannelForAsync(channelId));

vt1.join(); // 기다리기 
vt2.join();

**[리팩토링]**
List<Thread> threads = List.of(
        Thread.startVirtualThread(() -> isExistUserForAsync(userId)),
        Thread.startVirtualThread(() -> isExistChannelForAsync(channelId))
);
for (Thread thread : threads) {
    thread.join();
}

// 반환 Thread는 가상스레드다.
```

- 이 방식이 더 직관적이고 메모리 효율적

### V3 - Callable 사용(반환값이 필요한 경우)

이 예시는 ㄱ

```java
    private final ExecutorService ves =  Executors.newVirtualThreadPerTaskExecutor();
    
        Callable<Integer> task1 = () -> {
            isExistUserForAsync(userId);
            return 3;
        };
        Callable<Integer> task2 = () -> {
            isExistChannelForAsync(channelId);
            return 32;
        };

        // ves.invokeAll(List.of(task1, task2));

        Future<Integer> future1 = ves.submit(task1);
        Future<Integer> future2 = ves.submit(task2);
        Integer result1 = future1.get();
        Integer result2 = future2.get();
        System.out.println(result1 + result2);
        
[리팩토링]

Future<Integer> future1 = ves.submit(() -> {
    isExistUserForAsync(userId);
    return 3;
});
Future<Integer> future2 = ves.submit(() -> {
    isExistChannelForAsync(channelId);
    return 32;
});
Integer result1 = future1.get();
Integer result2 = future2.get();
System.out.println(result1 + result2);        
```

> **ExecutorService를 이용해 Callable 작업을 제출하는 방식**
>

(invokeAll은 그냥 넣어봤음)

- **반환값 관리**가 용이하고,
- **에러 처리**나 **타임아웃** 설정 등 다양한 옵션을 쉽게 적용할 수 있어서 일반적으로 선호

### V4 - CompletableFutre vs  Future

```java
    private final ExecutorService ves = Executors.newVirtualThreadPerTaskExecutor();
**

[Future 방법1 - 이게 그냥 깔끔한듯]**
Future<?> future1 = ves.submit(() -> isExistUserForAsync(userId));
Future<?> future2 = ves.submit(() -> isExistChannelForAsync(channelId));
try {
    future1.get();
    future2.get();
} catch (ExecutionException e) {

**[Future 방법2]**
List<Future<?>> futures = List.of(
    ves.submit(() -> isExistUserForAsync(userId)),
    ves.submit(() -> isExistChannelForAsync(channelId))
);

for (Future<?> future : futures) {
    future.get();
}

**[CompletableFuture를 쓰는 방법]**
try {
    CompletableFuture.allOf(
        CompletableFuture.runAsync(() -> isExistUserForAsync(userId), ves),
        CompletableFuture.runAsync(() -> isExistChannelForAsync(channelId), ves)
    ).join();
} catch (CompletionException e) {
```

- 가상 쓰레드가 비동기 지원해서 runAsync와 코드가 겹칠 수 있지만 깔끔해 보이긴 함
- But 가상 쓰레드가 있는데 굳이 CompletableFuture를 공부하기에는 시간이 부족해서 그냥 가상 스레드 방법쓰자
- 참고로 Future<?> 은 어떤 타입의 값이든 반환할 수 있는 Future 객체라는 뜻
    - get() 호출 시 Object처리됨

## 비동기 처리와 작업 완료 순서가 예측 불가능한 이유

> 동시성 작업이 시스템에 미치는 영향을 살펴보면서, 시간 순서가 보장되지 않는 점을 이해해보자
>

```java
public static void main(String[] args) throws Exception{
  ExecutorService vtExecutor = Executors.newVirtualThreadPerTaskExecutor();

  IntStream.range(0, 10).forEach(i ->
          vtExecutor.submit(() -> {
              sleep(1000);
              System.out.println("요청 완료 : " + i);
          })
  );
```

**착각**

- 비동기 처리가 이루어진다고해도 forEach에서 0~9까지 각 i값을 갖는 작업을 순차적으로 제출하니까 **먼저 제출한 요청이 먼저 처리되지 않을까????**
- **NO**

**결과표**

```java
요청 완료 : 2
요청 완료 : 9
요청 완료 : 5
요청 완료 : 1
요청 완료 : 4
요청 완료 : 6
요청 완료 : 0
요청 완료 : 3
요청 완료 : 8
요청 완료 : 7
```

0부터 9까지 submit이 호출되는 순서는 일정

- 근데?? **결과가 뒤죽박죽**이다
- 비동기로 실행되는 작업들은 각각 별도의 스레드(여기서는 가상 스레드)에서 동시 실행되므로, **실행 순서나 완료 순서는 예측할 수 없다**
- 예로, 0번 작업을 가장 먼저 제출했더라도 **“CPU 스케줄링”**이나 **“기타 요인”**에 의해 9번 인덱스의 작업이 더 빨리 처리될 수 있다.
    - **`submit()`을 호출한다고 즉시 실행되는 것이 아니라, 스레드 스케줄러가 이를 실행해야 하기 때문(운영체제의 스케쥴링에 따라 실행흐름 결정됨)**
    - 작업이 **어떤 CPU 코어에서 실행되는지??**
    - 현재 시스템이 **얼마나 바쁜지??**
    - **I/O 대기시간, 캐시 히트율, 메모리 상태** 등의 다양한 요인

> **결론**
>
> - 비동기 작업은 실행 순서를 보장하지 않는다.
> - **운영체제의 CPU 스케줄링과 여러 요인에 의해 실행 순서가 달라질 수 있다**

## 주의할 점

### 잘못된 사용 1

```java
    public static void main(String[] args) throws Exception{
        ExecutorService vtExecutor = Executors.newVirtualThreadPerTaskExecutor();

        IntStream.range(0, 10).forEach(i ->
                vtExecutor.submit(() -> {
                    sleep(1000);
                    System.out.println("요청 완료 : " + i);
                })
        );
        
        // 아래의 코드 누락 
        // vtExecutor.shutdown(); // 작업을 모두 제출한 후 shutdown() 호출
        // vtExecutor.awaitTermination(Long.MAX_VALUE, java.util.concurrent.TimeUnit.MILLISECONDS); // 모든 작업이 끝날 때까지 대기
    }
```

- `ExecutorService`는 `submit()`을 통해 작업을 스케줄링하지만, 메인 스레드가 종료되면 작업이 실행되지 않은 채 프로그램이 종료될 수 있음.

### 올바른 사용 1

```java
    public static void main(String[] args) throws Exception{
        ExecutorService vtExecutor = Executors.newVirtualThreadPerTaskExecutor();

        IntStream.range(0, 10).forEach(i ->
                vtExecutor.submit(() -> {
                    sleep(1000);
                    System.out.println("요청 완료 : " + i);
                })
        );

        vtExecutor.shutdown();
        vtExecutor.awaitTermination(Long.MAX_VALUE, java.util.concurrent.TimeUnit.DAYS);
    }

요청 완료 : 3
요청 완료 : 0
요청 완료 : 2
요청 완료 : 1
요청 완료 : 4
요청 완료 : 5
요청 완료 : 6
요청 완료 : 7
요청 완료 : 8
요청 완료 : 9
```

- 결과값은 순서대로 되는게 아니다

### 잘못된 사용 2

```java
try {
    Callable<?> submit1 = () -> {profile.deleteById(ProfileImgId()); return null;};
    Callable<?> submit2 = () -> {userStatusService.deleteByUserId(userId); return null;};
    Callable<?> submit3 = () -> {userRepository.delete(userId); return null;};
    ves.invokeAll(List.of(submit1, submit2, submit3));

```

![image.png](attachment:47fb7a58-26df-4067-a4e8-2a66c52725dd:image.png)

- 시도 : List.of로 Callable 리스트를 만들고 invokeAll에 넣는다
- `invokeAll()`은 Callable 작업들을 넣으면 동시에 병렬 실행
- `invokeAll()`은 **모든 작업이 완료될 때까지 블로킹됨**

### 올바른 시도2

```java

Callable<**Void**> submit1 = () -> {profile.deleteById(ProfileImgId()); return null;};
Callable<**Void**> submit2 = () -> {userStatusService.deleteByUserId(userId); return null;};
Callable<**Void**> submit3 = () -> {userRepository.delete(userId); return null;};
ves.invokeAll(List.of(submit1, submit2, submit3));
```

> `invokeAll(Collection<? extends Callable<T>> tasks)` 메서드
>
- 하나의 통일된 타입 `T`에 대해 `Callable<T>`를 여러 개 받아들이는 구조
- Void로 통합해서 한번에 넣는다

### 오해

```java
// 가상스레드 생성
private final ExecutorService ves = Executors.newVirtualThreadPerTaskExecutor();

try {
    Future<?> future1 = ves.submit(() -> isExistUserForAsync(userId));
    Future<?> future2 = ves.submit(() -> isExistChannelForAsync(channelId));

		// 가상 스레드에서는 get() 호출이 블로킹이지만 OS 스레드를 점유하지 않음.
    // get()에서 컨텍스트 스위칭 비용이 낮고,
    // 블로킹 시 자동으로 다른 가상 스레드를 실행할 수 있으므로, 성능 굿
    future1.get();  // 블로킹 발생 
    future2.get();  // future1이 끝난 후 실행
```

**기존 플렛폼 스레드였다면**

- future1 작업에서 블라킹이 발생하면 os스레드를 점유한 채 기다렸다가 future2이 진행된다.
- 이로 인해, OS 스레드가 **아무것도 하지 못하고 대기하고 스레드가 낭비**

**가상 스레드 사용 시**

- 가상 스레드는 **블로킹 시 플랫폼 스레드를 점유하지 않음**
- `future1.get();` 호출 → `future1`이 완료될 때까지 **해당 가상 스레드는 블로킹됨.**
- 하지만 가상 스레드는 **JVM이 자동으로 스케줄링**하기 때문에, **대기 중인 플랫폼 스레드를 해제하고 다른 가상 스레드를 실행**할 수 있음.
- 즉, `future1.get();`을 기다리는 동안 **OS 스레드가 놀지 않고 `future2`의 작업을 수행할 수 있음.**
- `future1`이 끝나면 다시 원래 실행되던 가상 스레드가 이어서 실행됨.

**가상 스레드 환경에서의 실행 흐름**

```scss

[메인 스레드] → future1.get() 호출 (I/O 블로킹 발생)
[OS 스레드]  → 대기하지 않고 future2.get() 실행 (다른 가상 스레드 처리)
[메인 스레드] → future1 완료 후 다시 실행
```

즉, **future1 가상 스레드가 블로킹되더라도 OS 스레드를 점유하지 않으므로 OS스레드는 다른 작업(futre2)이 수행될 수 있음. (**메인 스레드는 블로킹 상태로 있고)

✅**가상 스레드는 블로킹이 발생해도 OS 스레드를 점유하지 않음.**

✅**그렇기 때문에 비동기 처리 없이도 높은 동시성을 유지할 수 있음.**

✅**결과적으로 CompletableFuture를 사용하지 않고도 블로킹 스타일의 코드를 유지하면서 높은 성능을 낼 수 있음.**

Spring에서는 기본적으로 **가상 스레드 기반 `ScheduledTaskExecutor`를 제공하지 않음**.

## **가상 스레드를 언제 사용해야 할까?**

**✔ 사용하면 좋은 경우**

- **HTTP 요청을 많이** 처리하는 **웹 서버**
- 데이터베이스 쿼리, 파일 I/O 등 **블로킹 작업이 많은 경우**
- **높은 동시성(Concurrency)이 필요한** 서버 애플리케이션

**❌ 사용하면 성능이 떨어질 수 있는 경우**

- **CPU 연산**이 많은 **멀티스레드 연산 작업** (예: 이미지/영상 처리, 빅데이터 분석)
- **코어 개수에 최적화된 작업** (고정된 개수의 스레드가 효율적인 경우)
- 기존 블로킹 I/O 기반 라이브러리 사용 시 (라이브러리가 가상 스레드를 지원하는지 확인 필요)

vt1.join();
vt2.join();

Thread vt1 = Thread.startVirtualThread(() -> isExistUserForAsync(channelId));
Thread vt2 = Thread.startVirtualThread(() -> isExistChannelForAsync(channelId));