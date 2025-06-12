


우선 Message를 보내기 위해 RabbitTemplate을 만들어야한다
RabbitTemplate은 Auto-Config가 되어 있기에 거기에 내가 만든 Queue, Exchange, Binding등을 Inject해주면 된다.

```java

```


convertAndSend( )
- 파라미터 종류가 굉장히 다양하다
- 이번 예시에서 쓴 파라미터는  (exchange, routingKey, Object)이다
	- Queue에 메시지를 보내기 위해 exchange랑 routingKey정보가 필요하기 때문



> 컨트롤러로 보내고 확인해보기 