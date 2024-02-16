
### nacos 服务

```java
@EnableDiscoveryClient  
@SpringBootApplication  
public class GatewayApplication {  
  
    public static void main(String[] args) {  
        SpringApplication.run(GatewayApplication.class, args);  
    }  
  
}

```


## yml配置

```yml
server:  
  port: 10000  
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
      
```

