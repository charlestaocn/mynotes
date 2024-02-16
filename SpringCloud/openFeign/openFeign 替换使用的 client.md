

### 默认的是 jdk 的 HttpUrlConnection



## 1. 使用 httpclient

#### 添加依赖

<span style="color:pink">添加httpclient替换原本的</span>

```xml

<dependency>  
    <groupId>io.github.openfeign</groupId>  
    <artifactId>feign-httpclient</artifactId>  
</dependency>

```


<span style="color:pink">使用配置关闭或开启</span>

```yml
feign:  
  httpclient:  
    enabled: true
```

### 2. 使用okhttp

```xml
<dependency>  
    <groupId>io.github.openfeign</groupId>  
    <artifactId>feign-okhttp</artifactId>  
</dependency>
```

```yml
feign:  
  okhttp:  
    enabled: true
```