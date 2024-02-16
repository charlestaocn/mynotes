### <span style="color:pink">默认 jdk  的  HttpUrlConnection 发送请求 </span>
## pom依赖

```xml

<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-openfeign</artifactId>  
</dependency>

```


### 开启启动注解

```java

@EnableFeignClients  
@EnableDiscoveryClient  
@SpringBootApplication  
public class Service1Application {  
  
    public static void main(String[] args) {  
        SpringApplication.run(Service1Application.class, args);  
    }  
  
}

```


### service 动态代理的接口

```java
@FeignClient(name = "service2")  //服务名称
public interface Service2FeignClient {  

	//具体接口
    @RequestMapping("/test/responseAnyRequest?param=testFeignFromService1")  
    String feignService2();  
  
}
```

### 调用service 
```java


@Autowired  
private Service2FeignClient client;

@RequestMapping("send")  
public String send() {  
    String s = client.feignService2();  
    return "success";  
}

```