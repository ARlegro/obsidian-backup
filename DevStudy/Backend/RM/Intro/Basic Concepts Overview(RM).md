#Producer #Consumer #Queue #Exchange #Binding #Routing_Key #Message




![[Pasted image 20250612144016.png]]




| 용어                      | 내용                                                                                                                      |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Producer(생산자)           | 메시지를 생성하고 Consumer에게 보내기 위해 메시지 브로커를 통해 "간접적"으로 전송하는 구성요소이다                                                             |
| Message(메시지)            | Producer가 전송하는 "데이터 덩어리"<br>일반적으로 **Byte Stream으로 전달**된다.<br>즉, 단순히 텍스트로 전달되는 것이 아니다.<br>구성 : Header, Body  (like HTTP) |
| Message Broker(메시지 브로커) | 메시지를 생성하고 소비하는 클라이언트 간에 메시지를 전달하는 미들웨어<br>ex. Kafka, RabbitMQ 등                                                         |
| Exchange(교환기)           | 메시지를 수신하고 처리할 대상을 결정하는 구성요소<br>**메시지를 어느 Queue로 보낼지 결정하는 라우팅 에이전트(routing agent)** 역할                                   |
| Consumer(소비자)           | 메시지 브로커에서 메시지를 가져와서 ‘수신’하고 ‘처리’를 하는 주체                                                                                  |
| Binding(바인딩)            | 클라이언트 애플리케이션과 메시징 시스템 간의 연결을 하는 과정                                                                                      |
| Queue                   | 메시지 브로커 내에 메시지를 저장하는 저장소 or 버퍼 <br>송신자가 메시지를 전송하면 이를 수신자가 처리할때까지 대기시킵니다                                                 |
| Routing Key             | 메시지를 어떤 Queue에 라우팅할지 결정하기 위해 사용된다<br>Producer가 메시지를 보낼 때 Routing Key와 함께 보낸다                                            |


### 몇 개 팁 

#### 1. 하나의 Queue에 여러 Consumer가 있을 수 있다.
![[Pasted image 20250612144759.png]]



#### 2. Exchange 개념 다시보기 

 >메시지를 수신하고 처리할 대상을 결정하는 역할 (수신 + 라우팅 에이전트)



- **Producer와 Queue의 중개자 역할**
	- Producer가 메시지를 RabbitMQ로 보낼 때, 메시지는 Queue에 직접 전달되는 것이 아니라 반드시 **Exchange**로 먼저 전달된ㄷ다.
	- Exchange는 이 메시지를 받아서 자신의 타입과 메시지에 포함된 라우팅 키(routing key)에 따라 하나 이상의 큐로 메시지를 전달
- **주요 역할**
	- 메시지 수신 : 생산자로부터 메시지를 받는다.
	- 라우팅 : 메시지를 어떤 큐로 보낼지 결정 
	- 바인딩 : 큐사이의 연결관계를 정의한다.
	  
![[Pasted image 20250612145106.png]]


| Exchange 유형                   | 설명  |
| ----------------------------- | --- |
| **RabbitMQ Direct Exchange**  |     |
| **RabbitMQ Fanout Exchange**  |     |
| **RabbitMQ Topic Exchange**   |     |
| **RabbitMQ Headers Exchange** |     |
https://adjh54.tistory.com/497#2.%20Exchange%20%EC%9C%A0%ED%98%95-1-2
