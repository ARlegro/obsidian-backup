
> 알아볼 것 
> 1. JSON 타입으로 메시지 보내기
> 2. Multiple Queue이용하기 

![[Pasted image 20250612171312.png]]



일단 직렬화/역직렬화할 수 있는 POJO객체가 필요하다 

#### Queue-Binding 재설정 

✅JSON 전용 Queue 생성
```JAVA 
@Bean  
public Queue jsonQueue() {  
  return new Queue(rabbitMQProperties.queue_name_json());  
}

@Bean  
public Binding jsonBinding() {  
  return BindingBuilder  
      .bind(jsonQueue())  
      .to(exchange())  
      .with(rabbitMQProperties.routing_key_json());  
```

#### Message Converter, Template 커스텀 
> JSON을 직렬화 역직렬화 할 수 있게 등록 

✅Message Converter 빈 등록 - amqp 버전 
```java 
import org.springframework.amqp.support.converter.MessageConverter;

@Bean  
public MessageConverter converter() {  
		return new Jackson2JsonMessageConverter();  
}
```

> Note : MessageConverter는 amqp버전으로 import해야한다!!! 


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

#### Producer-Consumer 
```JAVA
public void sendJsonMessage(CustomerDTO customerDTO) {  
  log.info("보낼 JSON : {} ", customerDTO);  
  rabbitTemplate.convertAndSend(  
      rabbitMQProperties.exchange_name(),  
      rabbitMQProperties.routing_key_json(),  
      customerDTO  
  );

@RabbitListener(queues = {"${rabbitmq.queue-name-json}"})  
public void consumeJsonMessage(CustomerDTO message) {  
  log.info("받은 JSON 메시지 : {}", message);  
}


## JSON 으로 메시지 해보기 


```

### 테스트 


컨트롤러 
```JAVA 
@PostMapping("/publish")  
public ResponseEntity<String> sendJsonMessage(@RequestBody @Validated CustomerDTO message) {  
		rabbitMQProducer.sendJsonMessage(message);  
		return ResponseEntity.status(HttpStatus.OK).body("JSON-Message sent to R.M");  
}  
```
```JSON
{
    "id": 1,
    "name" : "yohan"
}
```

```bash
[rabbitmq-study] [nio-8080-exec-4] s.rabbitmq.producer.RabbitMQProducer     
: 보낼 JSON : CustomerDTO[id=1, name=yohan] 
[rabbitmq-study] [ntContainer#1-1] s.rabbitmq.consumer.RabbitMQConsumer     
: 받은 JSON 메시지 : CustomerDTO[id=1, name=yohan]
```


