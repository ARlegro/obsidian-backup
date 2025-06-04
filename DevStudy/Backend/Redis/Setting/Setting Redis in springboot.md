
### 환경 세팅 
```plain text

dependencies {
  
  implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}

```


```yaml




spring:
	...
	datasource:  
	  url: jdbc:mysql://${DB_HOST:host.docker.internal}:${DB_PORT:3307}/${DB_NAME:mydb}  
	  username: ${MYSQL_USERNAME:root}  
	  password: ${MYSQL_PASSWORD:pwd1234}  
	  driver-class-name: com.mysql.cj.jdbc.Driver

	data:
		redis:
			host: localhost
			prot: 6379

loggin:
	level:
		org.springframework.cache: trace
```

### Config 설정 

```java
@Configuration
public class RedisConfig {
  @Value("${spring.data.redis.host}")
  private String host;

  @Value("${spring.data.redis.port}")
  private int port;

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {
    // Lettuce라는 라이브러리를 활용해 Redis 연결을 관리하는 객체를 생성하고
    // Redis 서버에 대한 정보(host, port)를 설정한다. 
    return new LettuceConnectionFactory(new RedisStandaloneConfiguration(host, port));
  }
}
```

**RedisCacheConfig**
```java
@Configuration
@EnableCaching // Spring Boot의 캐싱 설정을 활성화
public class RedisCacheConfig {
  @Bean
  public CacheManager boardCacheManager(RedisConnectionFactory redisConnectionFactory) {
    RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration
        .defaultCacheConfig()
	      // Redis에 Key를 저장할 때 String으로 직렬화(변환)해서 저장
        .serializeKeysWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new StringRedisSerializer()))
        // Redis에 Value를 저장할 때 Json으로 직렬화(변환)해서 저장
        .serializeValuesWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new Jackson2JsonRedisSerializer<Object>(Object.class)
            )
        )
        // 데이터의 만료기간(TTL) 설정
        .entryTtl(Duration.ofMinutes(1L));

    return RedisCacheManager
        .RedisCacheManagerBuilder
        .fromConnectionFactory(redisConnectionFactory)
        .cacheDefaults(redisCacheConfiguration)
        .build();
  }
}

```

### 로직 
```java
@Cacheable(cacheNames = "getBoards", key = "'boards:page:' + #page + ':size:' + #size", cacheManager = "boardCacheManager") public List<Board> getBoards(int page, int size) {

		Pageable pageable = PageRequest.of(page - 1, size); 
		Page<Board> pageOfBoards = boardRepository.findAllByOrderByCreatedAtDesc(pageable); 
		return pageOfBoards.getContent(); 
}
```
- **속성 값** 
	- `cacheNames` : 캐시 이름을 설정
	- `key` : Redis에 저장할 Key의 이름을 설정
	- `cacheManager` : 사용할 `cacheManager`의 Bean 이름을 지정
	

>[!tip] @Cacheable 
>- Cache Aside 전략으로 캐싱이 적용
>- 즉, 요청이 들어오면 Redis 먼저 확인하는 것 
>- ![[Pasted image 20250604221421.png]]


