
## Pom依赖

```xml

<java.version>1.8</java.version>
<spring-cloud.version>Hoxton.SR9</spring-cloud.version>  
<spring-cloud-alibaba.version>2.2.6.RELEASE</spring-cloud-alibaba.version>


<dependency>  
    <groupId>com.alibaba.cloud</groupId>  
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>  
</dependency>


<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-dependencies</artifactId>  
    <version>${spring-cloud.version}</version>  
    <type>pom</type>  
    <scope>import</scope>  
</dependency>  
  
<dependency>  
    <groupId>com.alibaba.cloud</groupId>  
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>  
    <version>${spring-cloud-alibaba.version}</version>  
    <type>pom</type>  
    <scope>import</scope>  
</dependency>


```


## Yaml配置

```yml
spring:  
  cloud:  
    nacos:  
      discovery:  
        #nacos-server服务器地址  
        server-addr: localhost:8848  
        #当前的这个服务名称  
        service: service2
```

## springboot启动注解

```java
//开启client 发现
@EnableDiscoveryClient  
@SpringBootApplication  
public class Service2Application {  
  
    public static void main(String[] args) {  
        SpringApplication.run(Service2Application.class, args);  
    }  
  
}
```

