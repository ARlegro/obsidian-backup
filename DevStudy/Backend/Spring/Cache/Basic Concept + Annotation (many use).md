
Spring은 일부 데이터를 미리 메모리 저장소에 저장하고 저장된 데이터를 다시 읽어 사용하는 캐시 기능을 제공
 AOP를 사용하여 캐시 기능을 구현
 **캐시 데이터를 관리하는 기능은 별도의 캐시 프레임워크에 위임**한다.


### Cache Manager
캐시 추상화에서는 캐시 기술을 지원하는 캐시 매니저를 Bean으로 등록해야 한다.

Spring에서 제공하는 Cache Manager가 있지만, 나는 Redis의 Manager를 사용할거
Redis Cache Manager 빈 등록 [[Setting Redis in springboot]]
```java
@Configuration  
@EnableCaching  
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
```

## 애노테이션 

### @Cacheable 

#### 간단한 예시 
computePiDecimal 메서드가 잠재적으로 비용이 많이 드는 작업이라고 생각하자.
이럴 경우, 캐싱을 사용하는게 좋다.
```java 
@Component public class MyMathService {

		@Cacheable("piDecimals")
		public int computePiDecimal(int precision) {
		 ... 
		 }
```
- @Cacheable이 있으면, computePiDecimal메서드가 호출되기 전에
- Spring Cache 추상화는 `piDecimals`라는 캐시에서 `precision` 인자와 일치하는 항목을 찾는다.
- **만약 캐시에서 해당 항목을 찾으면,**
	- 캐시의 내용이 호출자에게 즉시 반환되며, 실제 메서드는 실행되지 않는다. 
- **만약 캐시에서 항목을 찾지 못하면,**
	- 메서드가 호출되고, 그 결과값이 캐시에 저장된 후 호출자에게 반환

#### 속성 
@Cacheable은 다음과 같은 속성을 가졌다.
```java
@Target({ElementType.TYPE, ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Inherited  
@Documented  
@Reflective  
public @interface Cacheable {  
    @AliasFor("cacheNames")  
    String[] value() default {};  
  
    @AliasFor("value")  
    String[] cacheNames() default {};  
  
    String key() default "";  
  
    String keyGenerator() default "";  
  
    String cacheManager() default "";  
  
    String cacheResolver() default "";  
  
    String condition() default "";  
  
    String unless() default "";  
  
    boolean sync() default false;
```
이 중 많이 쓰이는 거 위주로 
- **`value()` 또는 `cacheNames()`**:
	- **설명**: 이 메서드의 결과를 저장할 하나 이상의 캐시 이름을 지정합니다. 캐시는 여러 개를 지정할 수 있으며, 이 경우 모든 지정된 캐시에 결과가 저장됩니다.
	- ex. `@Cacheable("piDecimals")` 또는 `@Cacheable(value = {"piDecimals", "myOtherCache"})`
	- `value`와 `cacheNames`는 서로 **별칭(alias)** 관계입니다. **둘 중 하나만 사용해도** 됨 
- **`key()`**:
	- **설명**: 캐시 엔트리의 **키(key)를 명시적으로 생성**하기 위한 **SpEL** 표현식을 지정
	```java 
	@Cacheable(value = "users", key = "#userId"
	@Cacheable(value = "products", key = "#product.id")
  @Cacheable(value = "orders", key = "'order_' + #orderId")` 
  문자열 리터럴 + 매개변수를 조합
	```

- **`condition()`**:
	- 캐시 적용 여부를 결정하는 SpEL 표현식을 지정. 
	- 이 표현식이 `true`로 평가될 때만 캐싱이 적용
	- Example  
		- `@Cacheable(value = "results", condition = "#precision > 0")`: `precision`이 0보다 클 때만 캐싱
		- `@Cacheable(value = "users", condition = "#result != null")`: 메서드 결과가 `null`이 아닐 때만 캐싱합니다. (메서드 실행 후 조건 검사)

- **cacheManager**
	- 이 캐시 작업을 관리할 **`CacheManager` 빈의 이름**을 지정.
	- 여러 `CacheManager`를 구성했을 때 특정 캐시 매니저를 선택적으로 사용하고자 할 때 활용
	- **By Default** : Spring Boot가 자동 구성한 기본 `CacheManager`가 사용 💢

- **sync() : 멀티스레드 환경에서 중요** ⭐
	- By default = `false`
	- **개념** ✅
		- 캐시된 메서드가 동시에 여러 스레드에서 호출될 때, **하나의 스레드만 실제 메서드를 실행하고 나머지 스레드는 그 결과를 기다리도록 할지(`true`)를 지정
	- **캐시 스탬피드 현상 💢**
		- 이는 '멀티스레드 + sync=false + 캐시미스 발생' 시 일어난다. 
		- 이 상황에서, 동시에 여러 스레드가 요청이 들어오면 모든 스레드가 이 메서드를 실행하게 된다. 이는 엄청나게 큰 비용이 든다.
	- **현상 해결** : `true`로 설정 시
		- Spring Cache 추상화는 캐시 미스 발생 시 **단 하나의 스레드만** 실제 메서드를 실행하도록 보장
		- 캐시에서 데이터를 가져오지 못했을 때 메서드를 실행하는 스레드가 해당 캐시 엔트리에 대한 **잠금을 획득하므로, 다른 스레드들은 대기** 시키게 한다.
		- 동시성은 줄겠지만, 불필요한 중복 실행을 막아준다.
		- 애초에 메서드에 캐시쓰는 이유가 대부분 조회이기 때문에 동시성 정도는 버려도 됨 

>[!tip] Redis를 사용하더라도 `@Cacheable` 메서드의 비즈니스 로직이 매우 비용이 많이 드는 작업이라면, `sync = true`를 사용하여 캐시 스탬피드 현상을 방지하는 것을 고려가능




### @CacheEvict

#### 개념 
- 목적 : **캐시 데이터를 캐시에서 제거**
- **주요 사용처**: 원본 데이터를 변경하거나 삭제하는 메서드에 이 애노테이션을 적용
- **동작 원리**
	1. **원본 데이터가 변경/삭제되면, 해당 캐시 항목을 캐시에서 삭제** 
	2. 이렇게 제거된 후, 나중에 관련 `@Cacheable` 애노테이션이 적용된 메서드가 실행되면 변경된 데이터를 다시 조회하여 캐시에 새롭게 저장하게 됩니다. 
	> 이를 통해 캐시의 데이터 정합성(Cosistency)을 유지할 수 있다.

#### 속성 
```JAVA 
@Reflective  
public @interface CacheEvict {  
    @AliasFor("cacheNames")  
    String[] value() default {};  
  
    @AliasFor("value")  
    String[] cacheNames() default {};  
  
    String key() default "";  
  
    String keyGenerator() default "";  
  
    String cacheManager() default "";  
  
    String cacheResolver() default "";  
  
    String condition() default "";  
  
    boolean allEntries() default false;  
  
    boolean beforeInvocation() default false;  
}
```
@Cacheable과 크게 다르지 않다.
단지 이 애노테이션은 캐시를 제거하는 목적의 속성들인 것 

**`value()` 또는 `cacheNames()`**
- **설명**: 캐시에서 제거할 항목이 속한 하나 이상의 캐시 이름을 지정. 여러 캐시를 지정할 수 있으며, 이 경우 지정된 모든 캐시에서 항목이 제거
- 사용 예시 
	- `@CacheEvict("users")` 또는 `@CacheEvict(value = {"users", "userPermissions"})`

**`key()`**
- **설명**: 캐시에서 제거할 특정 엔트리의 **키(key)**를 명시적으로 생성하기 위한 SpEL표현식을 지정

뭐... 다른 것도 @Cacheable과 거의 비슷 


**beforeInvocation()**
- 캐시 제거 작업을 메서드 호출 **전(before)에 수행할지**, 아니면 메서드 호출 **후(after)에 수행할지**를 결정
- 기본값 : false ➡ 메서드 성공적으로 실행된 후 캐시 제거 
- **`beforeInvocation = false` (기본값):**
    - **메서드가 성공적으로 완료된 후에 캐시가 제거된다.**
    - 만약 메서드 실행 중 예외가 발생하면 캐시 제거는 수행되지 않는다. 
    - 이는 메서드 실패 시 캐시가 불필요하게 제거되는 것을 방지
      
- **`beforeInvocation = true`:**
    - 메서드 호출 **전에** 캐시가 제거된다.
    - 메서드 실행이 실패하더라도 캐시는 이미 제거된 상태이므로, 원본 데이터가 변경되지 않았음에도 캐시는 비어있을 수 있다. 
    - 이는 일시적인 불일치를 야기할 수 있지만, 특정 시나리오(예: 메서드 실패 시 캐시가 오래된 데이터를 반환하는 것을 방지)에서 유용할 수 있다.

```java
@CacheEvict(cacheNames = "boards", key = "'board:page:1:size:10'", cacheManager = "boardCacheManager")
public void update(Long id) {   
    Board board = boardRepository.findById(id)  
            .orElseThrow(() -> new RuntimeException("게시글이 존재하지 않습니다."));  
    board.setTitle("update title");
```



### @CachePut


#### 개념 
>[!tip] Modify작업 후 캐싱된 데이터를 갱신해주는 역할
>- `@CacheEvicet` ➡ 캐시 무효화 
>- `@CachePut` ➡ 캐시 갱신
- **목적** : 결과 값을 캐시에 갱신하는 것이 주 목적 
	- **원본 데이터의 변경과 동시에 캐시의 최신성을 보장**하여, 이후의 조회 요청이 항상 올바른 데이터를 캐시에서 가져갈 수 있도록 돕는 역할
	- 캐시 데이터의 정합성을 유지하면서 불필요한 DB 재조회 비용을 줄이는 데 목적
- 메서드는 항상 실행된다(캐시 여부랑 전혀 상관❌)
- **사용 경우** : 주로 데이터를 생성하거나 Modify하는 작업에서 사용된다.
#### 동작 방식 
1. 메서드 호출
2. 무조건 실제 메서드를 호출하여 실행
3. 실행 완료 후 결과 값을 지정한 캐시에 저장하거나 기존 값을 갱신 
> 즉, 메서드의 반환 값이 캐시에 저장 
	

### 💢자주 틀리는 실수 

>[!danger] SpEl쓸 때, 동적 변수 사용 시 **무조건 파라미터 값만 사용**해야한다.

#### 잘못된 예시 ❌
```java 
@Cacheable(cacheNames = "boards", key = "'board:page:'+ #page + ':size:' + #size", cacheManager = "boardCacheManager")  
public List<Board> getBoards(int page, int size) {  
    ...
}  
  
@CacheEvict(cacheNames = "boards", key = "'board:page:'+ #page + ':size:' + #size", cacheManager = "boardCacheManager")  
public void update(Long id) {  
    int page = 1;  
    int size = 10;  
    Board board = boardRepository.findById(id)  
        .orElseThrow(() -> new RuntimeException("게시글이 존재하지 않습니다."));  
    board.setContent("수정된 내용");  
}
```
> `#page`, `#size`같은 SpEl을 쓰는 거는 단순 변수로 안되고, 파라미터 값으로 넘어와야 적용된다.


