

gradle에 websocket, thymeleaf 추가 


Config 
```java 

@Configuration  
@RequiredArgsConstructor  
public class RabbitMQConfig {  
  
    private final RabbitMQConfigProperties properties;  
  
    @Bean  
    public Queue notificationQueue() {  
        return new Queue(properties.queueName(), false);  
    }  
  
    @Bean  
    public FanoutExchange notificationExchange() {  
        // 메시지 수신 시 연결된 모든 Queue에 전달  
        return new FanoutExchange(properties.fanoutExchange());  
    }  
  
    @Bean  
    public Binding notificationBinding() {  
        return BindingBuilder  
                .bind(notificationQueue())  
                .to(notificationExchange());  
    }
```

>[!tip] Fan-Out Exchange는 별도의 Routing-Key를 필요로하지 않는다.

publisher 
```java 

```

subscriber
```java
@Component  
@RequiredArgsConstructor  
public class NotificationSubscriber {  
  
    private final RabbitMQConfigProperties properties;  
    private final SimpMessagingTemplate simpMessageTemplate;  
  
    @Value("${rabbitmq.client.url}")  
    private String clientUrl;  
  
    @RabbitListener(queues = {"${rabbitmq.queue-name}"})  
    public void receive(String message) {  
        System.out.println("Received message: " + message);  
  
        // 특정 경로로 메시지 전송  
        simpMessageTemplate.convertAndSend(clientUrl, message);  
    }
```

> RabbitMQ로부터 수신한 메시지를 `simpMessageTemplate.convertAndSend(clientUrl, message)`를 통해 웹소켓(WebSocket)을 통해 연결된 클라이언트(사용자 브라우저 등)에게 실시간으로 전달

**`simpMessageTemplate`**
- **WebSocket 메시징 기능**과 관련된 객체
- **주요 역할** : STOMP(Simple Text-Oriented Messaging Protocol)를 통해 WebSocket 클라이언트(예: 웹 브라우저)에게 메시지를 전송하는 것
- `convertAndSend(destination, payload)`

이런 subscriber 패턴은 아래 같은 상황에서 자주 사용된다.
1. **실시간 알림** : 백엔드에서 생성된 알림을 웹소켓으로 사용자에게 즉시 푸시 
2. **채팅 어플리케이션** 
3. **실시간 대시보드** : 백엔드에서 생성되는 통계나 데이터 변화를 웹소켓을 통해 대시보드에 실시간으로 업데이트.




### Websocke Config 

```java
@Configuration  
@EnableWebSocketMessageBroker  
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {  
  
    @Override  
    public void configureMessageBroker(MessageBrokerRegistry registry) {  
        // 클라이언트 구독 경로  
        registry.enableSimpleBroker("/topic");  
        // 서버 발행 경로  
        registry.setApplicationDestinationPrefixes("/app");  
    }  
  
    @Override  
    public void registerStompEndpoints(StompEndpointRegistry registry) {  
        registry.addEndpoint("/ws").setAllowedOrigins("*").withSockJS();  
    }
```
- `@EnableWebSocketMessageBroker`: STOMP 기반 WebSocket 메시징을 활성화하겠다는 선언.
- `WebSocketMessageBrokerConfigurer`: WebSocket 관련 설정을 커스터마이징하기 위한 인터페이스.


- `addEndpoint("/ws")`: WebSocket 연결을 위한 엔드포인트 URL을 `/ws`로 지정.
- `withSockJS()`: **SockJS**를 사용해서 WebSocket을 지원하지 않는 브라우저도 fallback으로 연결 가능하게 함.


### 클라 index.html

```html
<div id="notifications"></div>  
  
<form id="notificationForm">  
    <input type="tel" id="notificationMessage" placeholder="입력하세요" required>  
    <button type="submit">전송</button>  
</form>  
<script>  
		// 1. SocketJS를 이용한 WebSocket 연결 설정
		// '/ws'는 백엔드 서버에서 WebSocket 연결을 수락하는 엔드포인트(URL)
    const socket = new SockJS('/ws'); // 서버의 WebSocket Endpoint 연결  
    
		// 2. STOMP 클라이언트 생성 
		// 
    const stompClient = Stomp.over(socket); // SockJS 객체를 STOMP 클라이언트로 wrapping

		// 3. STOMP 서버에 연결 
		// 서버와 STOMP 연결을 시도하고, 성공하면 콜백 함수를 실행
    stompClient.connect({}, function () {  
        console.log('Connected to WebSocket');  
        
        // 4. 메시지 구독 (Subscribe)
        // 클라이언트가 '/topic/notifications'라는 목적지(토픽)를 구독
        stompClient.subscribe('/topic/notifications', function (message) {  
            const notificationsDiv = document.getElementById('notifications');  
            const newNotification = document.createElement('div');  
            newNotification.textContent = message.body;  
            notificationsDiv.appendChild(newNotification);  
        });  

				// 서버로부터 메시지 고
        // 5. 서버로 전송도 가능  
        const form = document.getElementById('notificationForm')  
        form.addEventListener('submit', function (event) {  
            event.preventDefault();  
            const messageInput = document.getElementById('notificationMessage');  
            const message = messageInput.value;  
  
            stompClient.send('/app/send', {}, JSON.stringify({ message: message }));  
            messageInput.value = '';  
        })  
    });  
</script>  
</body>  
</html>
```

2. **STOMP 설정 이유  (Not only WebSocket)**
	- STOMP(Simple Text-Oriented Messaging Protocol)는 WebSocket 위에 계층화된 메시징 프로토콜
	- 일반 WebSocket은 바이너리/텍스트 데이터만 주고받는데, STOMP는 '헤더', '바디', '목적지(destination)'와 같은 개념을 도입하여 **메시지 교환을 더 구조화되고 쉽게 만든다.**
	- 이를 통해 클라이언트는 특정 '토픽'을 구독하거나 특정 '목적지'로 메시지를 보낼 수 있습니다.

4. **메시지 구독 (Subscribe) - `stompClient.subscribe('/topic/notifications ~`**
	- 클라이언트가 '/topic/notifications'라는 목적지(토픽)를 구독
	- 이는 백엔드 서버가 '/topic/notifications'로 메시지를 보내면, 이 클라이언트가 그 메시지를 수신하게 된다는 의미
	- 예: 백엔드에서 simpMessageTemplate.convertAndSend("/topic/notifications", "새로운 알림!")처럼 메시지를 보냈을 때 클라이언트가 받는다.
	- message 객체는 STOMP 메시지이며, 실제 내용은 message.body에 담겨 있다.
	  
5. **서버로 메시지 전송** 
	- 사용자가 폼을 제출할 때, **클라이언트에서 서버로 메시지를 전송**
	- **'/app/send'** : 백엔드 서버의 특정 엔드포인트(목적지)로 메시지를 보낸다
	- 예시 : Spring Boot 백엔드에서는 **`@MessageMapping("/send")` 어노테이션이 붙은 컨트롤러 메소드가 이 메시지를 받게 된다**.
		- {}: 전송 시 포함할 헤더 (여기서는 비어 있음). 
		- JSON.stringify({ message: message }): 전송할 메시지의 내용(payload). 여기서는 자바스크립트 객체를 JSON 문자열로 변환하여 보낸다. 
	- `messageInput.value = '';` : 메시지 전송 후 입력 필드 초기화 


>[!tip] 자바스크립트 코드 Flow 정리 
>1. 클라이언트 웹소켓 연결 
>2. 클라이언트 구독 - `stompClient.subscribe('/topic/notifications')`
>	- 백엔드의 Spring `SimpMessagingTemplate`이 `/topic/notifications` 목적지로 메시지를 `convertAndSend()` 할 때마다 해당 메시지를 받게된다.
>	- 백엔드의 `WebSocketConfig`에서는 `config.enableSimpleBroker("/topic")`과 같이 `/topic`으로 시작하는 목적지들이 브로커에 의해 처리되도록 설정되었을 것
>   
>3. 클라이언트 메시지 전송 
>	- @MessageMapping("/send) 가 붙은 메서드로 라우팅된다.


