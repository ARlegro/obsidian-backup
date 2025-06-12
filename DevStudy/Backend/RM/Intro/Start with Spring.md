
Spring은 AMQP프로토콜을 제공하며 RabbitMQ와 ??

```yaml
spring:  
  rabbitmq:  
    host: localhost  
    port: 5672  
    username: guest  
    password: guest
```
이건 기본값이다.


RabbitMQ플러그인 깔면 스프링부트를 실행할 때, RabbitMQ 서버와 Connect한다.
![[Pasted image 20250612160201.png]]


## Config 

앞서 배운 Core Concept들을 Spring에서 어떻게 Config하는지 알아볼 것 
[[Basic **Concepts** Overview]]


기본 Queue-Exchange-Binding Config
```java 
@Bean  
public Queue queue() {  
    return new Queue(queueName);  
}  
  
@Bean  
public TopicExchange exchange() {  
    return new TopicExchange(exchangeName);  
}  
  
// Routing Key Config 필요  
// Binding between Queue and Exchange using Routing Key  
@Bean  
public Binding routingKey() {  
    return BindingBuilder  
            .bind(queue())  
            .to(exchange())  
            .with(routingKey);  
}
```



이 외에도 ConnectionFactory, RabbitTemplate, RabbitAdmin도 Config할 수 있지만 이 3개는 Auto-Config 되는 대상들이라 따로 설정하지 않아도 된다.