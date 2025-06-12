
> 알아볼 것 
> 1. JSON 타입으로 메시지 보내기
> 2. Multiple Queue이용하기 

![[Pasted image 20250612171312.png]]





일단 직렬화/역직렬화할 수 있는 POJO객체가 필요하다 


✅JSON 전용 Queue 생성
```JAVA 

```


✅Json 바인딩 생성 



✅RabbitTemplate Config 
- JSON 메시지를 지원할 수 있도록 Config 해야한다.


✅Message Converter 빈 등록 - amqp 버전 
```java 


@Bean  
public MessageConverter converter() {  
		return new Jackson2JsonMessageConverter();  
}
```
- Jackson2JsonM.C도 amqp버전으로 


✅RabbitTemplate 커스텀
```java
@Bean  
public AmqpTemplate amqpTemplate(ConnectionFactory connectionFactory) {  
    RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);  
    rabbitTemplate.setMessageConverter(converter());  
    return rabbitTemplate;  
}
```
- AmqpTemplate : 인터페이스
- RabbitTemplate : 구현체
- ConnectionFactory도 amqp 버전 





## JSON 으로 메시지 해보기 


RabbitMQJsonProducer 클래스 생성



