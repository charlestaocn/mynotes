 
 ## 为nacos server  提供服务的客户端应用服务器


## [[nacos-client 配置]]


## 调用nacos-client 的服务

### 使用`RestTemplate`发送请求

```java

/**  
 * 注入 restTemplate  
 * LoadBalanced:使用负载均衡interceptor  
 * * @see org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor  
 */

@Bean  
@LoadBalanced  
public RestTemplate restTemplate() {  
    return new RestTemplate();  
}

```

###  `@LoadBalanced ` 注解作用

###  作用: 负载均衡并且解析 `serviceId =》 servicse`


```java

//package org.springframework.cloud.client.loadbalancer;


public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
	//RibbonLoadBalancerClient
	private LoadBalancerClient loadBalancer;
	//拦截restTemplate 请求 进行解析
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,  
	       final ClientHttpRequestExecution execution) throws IOException {  
	    final URI originalUri = request.getURI();  
	    String serviceName = originalUri.getHost();  
	    Assert.state(serviceName != null,  
	          "Request URI does not contain a valid hostname: " + originalUri);  
	    return this.loadBalancer.execute(serviceName,  
	          this.requestFactory.createRequest(request, body, execution));  
	}
}




```





### 使用 `DiscoveryClient` 请求`nacos-server` 获取 已注册的服务列表

```java

@Autowired  
private DiscoveryClient discoveryClient;
List<ServiceInstance> services = discoveryClient.getInstances("${serviceName}");


/**  
 * 按服务名(serviceId)请求(必须时 @LoadBalanced 注解过经过拦截器解析的 restTemplate 才行)  
 */
ResponseEntity<String> response = restTemplate.getForEntity("http://" + services.get(0).getServiceId() + "/test/responseAnyRequest?param=testFromSerivce1", String.class);


/**  
 * 按服务uri进行请求(不需要上面的interceptor进行服务名解析，能直接发送！)  
 */
ResponseEntity<String> response1 = restTemplate.getForEntity(services.get(0).getUri() + "/test/responseAnyRequest?param=testFromSerivce1", String.class);


```

### 负载均衡实现


####  1.  ribbon (已经停止更新维护了） 
<span style="color:pink"> nacos-discovery 包中带了  ribbon 的包！！！ </span>

[[SpringCloud/nacos/_resources/nacos-client/b755dc19f750a2a1eea1aa7412fce396_MD5.jpeg|Open: 截屏2023-11-11 18.04.13.png]]
![[SpringCloud/nacos/_resources/nacos-client/b755dc19f750a2a1eea1aa7412fce396_MD5.jpeg]]



```java
//package org.springframework.cloud.netflix.ribbon;

public class RibbonLoadBalancerClient implements LoadBalancerClient {
		//将serviceId 转换成 服务 再进行请求
		public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {  
		    ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);  
		    Server server = this.getServer(loadBalancer, hint);  
		    if (server == null) {  
		        throw new IllegalStateException("No instances available for " + serviceId);  
		    } else {  
		        RibbonServer ribbonServer = new RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));  
		        return this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request);  
		    }  
		}
}
```



### 2.   spring-cloud-loadbalance 

<span style="color:pink"> 两者都默认能使  @LoadBalanced 注解生效</span>

[[SpringCloud/nacos/_resources/nacos-client/56b406029d09d85e5e4acc2f166ad79c_MD5.jpeg|Open: 截屏2023-11-11 22.17.56.png]]
![[SpringCloud/nacos/_resources/nacos-client/56b406029d09d85e5e4acc2f166ad79c_MD5.jpeg]]