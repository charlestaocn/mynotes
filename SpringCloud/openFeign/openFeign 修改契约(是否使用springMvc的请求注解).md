
[[SpringCloud/openFeign/_resources/openFeign 修改契约(是否使用springMvc的请求注解)/808e823da83b854c77bd388409b6ccd4_MD5.jpeg|Open: 截屏2023-11-12 10.56.09.png]]
![[SpringCloud/openFeign/_resources/openFeign 修改契约(是否使用springMvc的请求注解)/808e823da83b854c77bd388409b6ccd4_MD5.jpeg]]


### 1.  默认使用 `feign.Contract.BaseContract` 进行请求注解解析发送

#### <span style="color:pink">  解析的SpringMvc 的注解</span>

```java

@FeignClient(name = "service2")  //服务名称
public interface Service2FeignClient {  

	//具体接口
	//springmvc 的 注解
    @RequestMapping("/test/responseAnyRequest?param=testFeignFromService1")  
    String feignService2();  
  
}

```

```java

feign.Contract.BaseContract

protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {  

//·····
  if (targetType.getInterfaces().length == 1) {  
    processAnnotationOnClass(data, targetType.getInterfaces()[0]);  
  }  
  processAnnotationOnClass(data, targetType);  
  
  for (final Annotation methodAnnotation : method.getAnnotations()) {  
    processAnnotationOnMethod(data, methodAnnotation, method);  
  }  
 
  //·····
  }

```


### 2. 修改使用 `Default(原生的注解)contract` 发送请求(有可能公司框架不使用mvc)

#### <span style="color:pink">  解析的原生的注解</span>

[[SpringCloud/openFeign/_resources/openFeign 修改契约(是否使用springMvc的请求注解)/35313fbaa634da26ee9c798c0b5730b2_MD5.jpeg|Open: 截屏2023-11-12 11.08.43.png]]
![[SpringCloud/openFeign/_resources/openFeign 修改契约(是否使用springMvc的请求注解)/35313fbaa634da26ee9c798c0b5730b2_MD5.jpeg]]


#### 修改配置


```java
public class FeignConfig {  
  
    @Bean  
    public Logger.Level feignLogger() {  
        return Logger.Level.FULL;  
    }  
  
    @Bean  
    public Contract feignContract(){  
        return new Contract.Default();  
    }  
  
}
```


#### 修改使用原生注解

```java
feign.Contract.Default

@FeignClient(name = "service2", configuration = FeignConfig.class)  
public interface Service2FeignClient {  
    @RequestLine("GET /test/responseAnyRequest?param=testFeignFromService1")  
    String feignService2();  
  
}

```

