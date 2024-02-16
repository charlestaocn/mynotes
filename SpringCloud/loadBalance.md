# 一、介绍

Spring Cloud LoadBalancer是Spring Cloud官方自己提供的客户端负载均衡器,抽象和实现，用来替代Ribbon（已经停更），

# 二、Ribbon和Loadbalance 对比

| **组件**            | **组件提供的负载策略**                                                                                                                                                                                          | **支持负载的客户端**     |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| Ribbon                    | 随机 RandomRule<br />轮询 RoundRobinRule    <br />重试 RetryRule<br />最低并发 BestAvailableRule<br />可用过滤 AvailabilityFilteringRule<br />响应时间加权重 ResponseTimeWeightedRule<br />区域权重 ZoneAvoidanceRule | Feign或openfeign、RestTemplate |
| Spring Cloud Loadbalancer | RandomLoadBalancer 随机（高版本有，此版本没有RoundRobinLoadBalancer 轮询（默认）                                                                                                                                      | Ribbon 所支持的、WebClient     |

LoadBalancer 的优势主要是，支持**响应式编程**的方式**异步访问**客户端，依赖 Spring Web Flux 实现客户端[负载均衡](https://so.csdn.net/so/search?q=负载均衡&spm=1001.2101.3001.7020)调用。

# 三、整合LoadBlance

> 注意如果是Hoxton之前的版本，默认负载均衡器为Ribbon，需要移除Ribbon引用和增加配置spring.cloud.loadbalancer.ribbon.enabled: false。

## 1、升级版本

| Spring Cloud Alibaba | Spring cloud            | Spring Boot   |
| -------------------- | ----------------------- | ------------- |
| 2.2.6.RELEASE        | Spring Cloud Hoxton.SR9 | 2.3.2.RELEASE |

## 2、移除ribbon依赖，增加loadBalance依赖

```pom
<!--nacos-服务注册发现-->
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
	<exclusions>
		<!--将ribbon排除-->
		<exclusion>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
		</exclusion>
	</exclusions>
</dependency>


<!--添加loadbalanncer依赖, 添加spring-cloud的依赖-->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

# 四、自定定义负载均衡器

```java
package com.msb.order.loadbalance;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.reactive.DefaultResponse;
import org.springframework.cloud.client.loadbalancer.reactive.EmptyResponse;
import org.springframework.cloud.client.loadbalancer.reactive.Request;
import org.springframework.cloud.client.loadbalancer.reactive.Response;
import org.springframework.cloud.loadbalancer.core.ReactorServiceInstanceLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import reactor.core.publisher.Mono;
import java.util.List;
import java.util.Random;

public class CustomRandomLoadBalancerClient implements ReactorServiceInstanceLoadBalancer {
 
    // 服务列表
    private ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
 
    public CustomRandomLoadBalancerClient(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider) {
        this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
    }
 
    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider.getIfAvailable();
        return supplier.get().next().map(this::getInstanceResponse);
    }
 
    /**
     * 使用随机数获取服务
     * @param instances
     * @return
     */
    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {
        System.out.println("进来了");
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }
 
        System.out.println("进行随机选取服务");
        // 随机算法
        int size = instances.size();
        Random random = new Random();
        ServiceInstance instance = instances.get(random.nextInt(size));
 
        return new DefaultResponse(instance);
    }
}
```

```java
@EnableDiscoveryClient
@SpringBootApplication
// 设置全局负载均衡器
@LoadBalancerClients(defaultConfiguration = {CustomRandomLoadBalancerClient.class})
// 指定具体服务用某个负载均衡
//@LoadBalancerClient(name = "msb-stock",configuration = CustomRandomLoadBalancerClient.class)
//@LoadBalancerClients(
//        value = {
//                @LoadBalancerClient(value = "msb-stock",configuration = CustomRandomLoadBalancerClient.class)
//        },defaultConfiguration = LoadBalancerClientConfiguration.class
//)
public class OrderApplication {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class);
    }
}
```

# 五、重试机制

```yaml
spring:
  cloud:
    loadbalancer:
      #以下配置为LoadBalancerProperties 配置类
      clients:
        #default 表示去全局配置，如要针对某个服务，则填写毒地应的服务名称即可
        default:
          retry:
            enbled: true
            #是否有的的请求都重试，false表示只有GET请求才重试
            retryOnAllOperation: true
            #同一个实例的重试次数，不包括第一次调用：比如第填写3 ，实际会调用4次
            maxRetriesOnSameServiceInstance: 3
            #其他实例的重试次数，多节点情况下使用
            maxRetriesOnNextServiceInstance: 0
```


