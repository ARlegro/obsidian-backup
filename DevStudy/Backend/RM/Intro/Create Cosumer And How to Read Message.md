



@RabbitListener 사용 

```java
@Slf4j  
@Service  
public class RabbitMQConsumer {  
      
    @RabbitListener(queues = {"${rabbitmq.queue.name}"}) // 이게 뭐지  
    public void consume(String message){  
        log.info("Message received -> {}", message);  
    }
```

