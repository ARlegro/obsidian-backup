DB에서 동시성 테스트할 때, K6말고 JUnit으로도 해보고 싶어서 공부 


### 개념 
- 하나 또는 여러 스레드가 특정 작업을 시작하거나 완료할 때까지 다른 스레드(들)의 실행을 기다리게 하는 데 사용E된다.
- 여러 스레드가 동시에 출발선에 서서 신호(카운트다운 완료)를 기다리다가 동시에 출발 가능


### 동작 방식 
초기화 시점에 **정수 형태의 카운트 값**을 가진다.
- 이 카운트 값은 "문이 열리기 위해 필요한 남은 작업 수" 또는 "출발 신호가 나기까지 남은 시간"이라고 생각할 수 있다.

주요 메서드
1. await()
	- 이 메서드를 호출하는 스레드는 **카운트 값이 0이 될 때까지 블록(대기)된다**
	- 카운트 값이 0이 되면, `await()`를 호출했던 모든 스레드는 블록이 해제되어 다음 작업을 진행할 수 있다.
	  
2. countDonw()
	- `CountDownLatch`의 **내부 카운트 값이 1 감소한다.**
	- 카운트 값이 0이 되면, `await()`에 의해 대기 중이던 모든 스레드가 동시에 해제된다.


### 활용 예시 
> 목표 : N개의 스레드가 준비 완료된 상태에서 동시에 작업을 시작하도록 만드는 테스트 


```java
public static void main(String[] args) throws InterruptedException {
		final int numberOfThreads = 5; // 동시에 실행할 스레드 수
		
		// 모든 스레드가 동시에 시작할 신호를 기다리는 래치
		// 카운트가 1이므로, 메인 스레드가 한 번만 countDown() 하면 모두 출발 가능
		final CountDownLatch startSignal = new CountDownLatch(1); 
		
		// 모든 스레드가 작업 완료를 알릴 때까지 메인 스레드가 기다리는 래치
		// numberOfThreads 만큼의 스레드가 각각 countDown() 해야 0이 됨
		final CountDownLatch finishSignal = new CountDownLatch(numberOfThreads); 

		ExecutorService executor = Executors.newFixedThreadPool(numberOfThreads);

		System.out.println("모든 스레드 준비 중...");

		for (int i = 0; i < numberOfThreads; i++) {
				final int threadId = i;
				executor.submit(() -> {
							
							// 모든 스레드는 startSignal이 0이 될 때까지 기다린ㄷ다. 
							startSignal.await(); // <- 여기서 모든 스레드가 블록됨
}
```

1. 출발 준비 래치 (준비 완료 확인)
	- CountDownLatch startSignal = new CountDownLatch(1);
	- 이 래치는 모든 스레드가 준비되었을 때 한 번의 `countDown()`으로 동시에 출발 신호를 주기 위해 사용되는거다.
	- 이 CountDownLatch의 값이 0이 되면 더 이상 `executor.submit`의 동작이 블락되지 않고 실행 됨 
	  
2. 모든 스레드 실행 완료 대기 래치 
	

