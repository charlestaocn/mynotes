
# spring data redis reactive (不推荐 麻烦) 

配合 webflux 使用

## 依赖
```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
implementation 'com.alibaba:fastjson:latest.release'
```

## 需要配置key value 序列化 默认jdk 的不行
```java
@Configuration  
public class RedisConfiguration {  
  
    @Bean  
    public ReactiveRedisTemplate<Object, Object> reactiveRedisTemplate(ReactiveRedisConnectionFactory factory) {  
        GenericFastJsonRedisSerializer stringFastJsonRedisSerializer = new GenericFastJsonRedisSerializer();  
        RedisSerializationContext<Object, Object> serializationContext = RedisSerializationContext  
                .newSerializationContext(stringFastJsonRedisSerializer)  
                .key(stringFastJsonRedisSerializer)  
                .build();  
        return new ReactiveRedisTemplate<>(factory, serializationContext);  
    }
}
```

## 使用

```java

private ReactiveRedisTemplate reactiveRedisTemplate;

@Autowired  
public void setReactiveRedisTemplate(ReactiveRedisTemplate reactiveRedisTemplate) {  
    this.reactiveRedisTemplate = reactiveRedisTemplate;  
}

public Mono<Boolean> insertUserIntoRedis(User user) {  
    HashMap<String, Object> map = new HashMap<>();  
    map.put("userName", user.getUserName());  
    map.put("password", user.getPassword());  
    map.put("depId", user.getDepId());  
    ReactiveHashOperations ops = reactiveRedisTemplate.opsForHash();  
    Mono<Boolean> put = ops.putAll(user.getUserName(), map);  
    return put;  
}  
  
  
public Mono<User> getUserInfoByUserName(String userName) {  
    ReactiveHashOperations ops = reactiveRedisTemplate.opsForHash();  
    Flux<Map.Entry<String, Object>> entries = ops.entries(userName);  
    return entries  
		    //可以使用工具类进行entry -> T.class
            .collectMap(Map.Entry::getKey, Map.Entry::getValue)  
            .map(userMap -> {  
                String jsonString = JSON.toJSONString(userMap);  
                return JSON.parseObject(jsonString, User.class);  
            });  
}

```

## 注意

==jackson 的 和fastjson 的 在序列 string  类型的值时 会添加 “” ==
[[redis/_resources/reactive redis/72d5dec2f0f328609c403c0a22bfe298_MD5.jpeg|Open: 截屏2024-02-04 21.55.07.png]]
![[redis/_resources/reactive redis/72d5dec2f0f328609c403c0a22bfe298_MD5.jpeg]]

## 总结: 没jedis 好用！

# redission 

```groovy 
implementation 'org.redisson:redisson-spring-boot-starter:3.20.0'
```

```java
public Config redissonConfig() {  
    Config config = new Config();  
    config.useSingleServer().setAddress("redis://" + host + ":6379");  
    return config;  
}  
  
@Bean  
public RedissonReactiveClient reactiveRedissonClient() {  
    RedissonClient redissonClient = Redisson.create(redissonConfig());  
    return redissonClient.reactive();  
}
```

```java
@Autowired  
private RedissonReactiveClient redissonReactiveClient;  
  
@Test  
public void testRedisson() {  
    RMapReactive<Object, Object> testMap = redissonReactiveClient.getMap("testRedisson");  
    Mono<Object> put = testMap.put("username", "password");  
    Mono.when(put).block();  
}
```

==redisson 存储没有“”==