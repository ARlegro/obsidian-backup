
```gradle
... 

dependencies {
  
  implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}

```


```yaml

spring:
	...
	data:
		redis:
			host: localhost
			prot: 6379

loggin:
	level:
		org.springframework.cache: trace
```


1.81s
49ms
9
9 
15
11
12

