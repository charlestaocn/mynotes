
### 全局修改请求属性


### 修改前

```java

//添加 header
@RequestMapping(value = "/test/responseAnyRequest?param=testFeignFromService1", headers = {"token=customToken"})  
String feignService2();

```

### 修改后

```java

//添加 header
@RequestMapping(value = "/test/responseAnyRequest?param=testFeignFromService1")  
String feignService2();

```

### 1.1 全局 interceptor

	全局请求都会经过

```java
@Component  
public class FeignInterceptor implements RequestInterceptor {  
  
    @Override  
    public void apply(RequestTemplate template) {  
        Map<String, Collection<String>> headers = new HashMap<>(); 
        //请求添加header 
        headers.put("token", Collections.singletonList("customToken"));  
        template.headers(headers);  
    }  
}
```

### 1.2 config 配置

```java

public class FeignConfig {  
    @Bean  
    public RequestInterceptor requestInterceptor() {  
        return new FeignInterceptor();  
    }  
}
```

##### 1.2.1 单独使用Config

	
```java
//只有使用config类的client才会经过interceptor 

@FeignClient(name = "service2", configuration = FeignConfig.class)  
public interface Service2FeignClient {  
  
    @RequestMapping(value = "/test/responseAnyRequest?param=testFeignFromService1")  
    String feignService2();  
  
}

```
