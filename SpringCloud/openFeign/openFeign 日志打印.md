

### yml 配置

```yml

logging:  
  level:  
    com.test: debug
    
```


### 注入方法


```java

@Bean  
public Logger.Level feignLogger() {  
    return Logger.Level.FULL;  
}

```


###  1. 在单 `ClientService `上使用
```java

@FeignClient(name = "service2", configuration = FeignConfig.class)

```

### 2. 在全局使用

#### 2.1 使用`Configuration`

```java

@Configuration  
public class FeignConfig {  
  
    @Bean  
    public Logger.Level feignLogger() {  
        return Logger.Level.FULL;  
    }  
  
}

```
#### 2.2 或者使用启动注解添加`defaulConfig`

```java
@EnableFeignClients(defaultConfiguration = FeignConfig.class)  
@EnableDiscoveryClient  
@SpringBootApplication  
public class Service1Application {  
    public static void main(String[] args) {  
        SpringApplication.run(Service1Application.class, args);  
    }  
}
```