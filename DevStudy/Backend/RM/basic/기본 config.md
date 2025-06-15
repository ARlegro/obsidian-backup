

```java 
@Bean  
public Queue queue() {  
    return new Queue(queueName);  
}

@Bean  
public Queue queue() {  
    return new Queue(queueName, false);   << 2번째 파라미터 = durable
}

```

2번째 파라미터 = durable
- false : 휘발성(volatile)
	- 서버가 종료되거나 시작될 때 Queue의 메시지가 사라지게 함 
- true : 영속성(persistent)



