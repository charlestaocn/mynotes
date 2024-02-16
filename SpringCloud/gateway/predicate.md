
## predicate 断言

```yml

routes:  
        - id: service2-route  
          uri: lb://service2  
          predicates:  
            - Path=/test/*  
            #token,123 => token=123  
#            - Header=token,123  
#            - Cookie=token,123  
#            - Between=2023-02-11T00:00:00+08:00[Asia.shanghai],2023-02-12T00:00:00+08:00[Asia.shanghai]

```

### 经过Gateway router 

```http
%% localhost:10000 => gateway 项目地址 %%
%% 
	uri: lb://service2  
	    predicates:  
	        - Path=/test/* 
	
%%
%% 
	test下的请求都会负载均衡的路由到 service2 服务上 
%%
GET  http://localhost:10000/test/responseAnyRequest?param=testFeignFromService1

```


### 自定义 `predicate`

自定义 请求 header predicate

```java
  
@Component  
public class AuthorizationRoutePredicateFactory extends AbstractRoutePredicateFactory<AuthorizationRoutePredicateFactory.Config> {  
  
    public AuthorizationRoutePredicateFactory() {  
        super(AuthorizationRoutePredicateFactory.Config.class);  
    }  
  
    @Override  
    public List<String> shortcutFieldOrder() {  
        return Arrays.asList("authorization");  
    }  
  
    @Override  
    public Predicate<ServerWebExchange> apply(AuthorizationRoutePredicateFactory.Config config) {  
        return exchange -> {  
            ServerHttpRequest request = exchange.getRequest();  
            return Optional.of(request.getHeaders()).map(  
                    headers ->  
                            Optional.ofNullable(headers.get(HttpHeaders.AUTHORIZATION))  
                                    .map(opTokens ->  
                                            opTokens.stream().filter(config.getAuthorization()::equals).findAny().orElse(null) != null  
  
                                    ).orElse(false)  
            ).orElse(false);  
        };  
    }  
  
    public static class Config {  
        private String authorization;  
  
        public String getAuthorization() {  
            return authorization;  
        }  
  
        public void setAuthorization(String authorization) {  
            this.authorization = authorization;  
        }  
    }  
}
```


```yml
spring:  
  cloud:  
    nacos:  
      discovery:  
        server-addr: localhost:8848  
        service: gateway  
    gateway:  
      discovery:  
        locator:  
          enabled: false  
      enabled: true  
      routes:  
        - id: service2-route  
          uri: lb://service2  
          predicates:  
            - Path=/test/*  
            - Authorization=123
```
```

```http

GET  http://localhost:10000/test/responseAnyRequest?param=testFeignFromService1  
#header
Authorization: 123

```