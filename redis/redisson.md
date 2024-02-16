##  配置
#### springboot 依赖

```java

implementation 'org.redisson:redisson-spring-boot-starter:3.19.1'

```

#### yml 配置

```yaml
spring:  
  data:  
    redis:  
      host: 127.0.0.1  
      port: 6379  

```

### configuration

```java


package org.playground.playground.config;  
  
import org.redisson.Redisson;  
import org.redisson.api.RedissonClient;  
import org.redisson.config.Config;  
import org.redisson.config.TransportMode;  
import org.redisson.spring.cache.CacheConfig;  
import org.redisson.spring.cache.RedissonSpringCacheManager;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.beans.factory.annotation.Value;  
import org.springframework.boot.autoconfigure.data.redis.RedisProperties;  
import org.springframework.cache.CacheManager;  
import org.springframework.cache.annotation.EnableCaching;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
  
import java.util.HashMap;  
import java.util.Map;  
  
/**  
 * @author charles tao  
 */
 @Configuration  
@EnableCaching  
public class RedissonConfiguration {  
    private RedisProperties redisProperties;  
  
  
    @Bean(destroyMethod = "shutdown")  
    public RedissonClient redissonClient() {  
        Config config = new Config();  
        config.setTransportMode(TransportMode.NIO);  
        config.useSingleServer().setAddress("redis://" + redisProperties.getHost() + ":" + redisProperties.getPort());  
        return Redisson.create(config);  
    }  
  
  
    /**  
     * redisson cache config     
     *     
     * @param ttl     time to live  
     * @param maxIdle maxIdleTime  
     * @return config  
     */    @Bean  
    public CacheConfig redissonCacheConfig(@Value("${redisson.cache.ttl}") Integer ttl, @Value("${redisson.cache.maxIdle}") Integer maxIdle) {  
        return new CacheConfig(ttl, maxIdle);  
    }  
  
  
    /**  
     * spring cache manager     
     *     
     * @param redissonClient redisson  
     * @param cacheConfig    cacheConfig  
     * @return manager  
     */    
     @Bean  
    public CacheManager cacheManager(RedissonClient redissonClient, CacheConfig cacheConfig) {  
        Map<String, CacheConfig> config = new HashMap<>();  
        config.put("cacheMap", cacheConfig);  
        return new RedissonSpringCacheManager(redissonClient, config);  
    }  
  
    @Autowired  
    public void setRedisProperties(RedisProperties redisProperties) {  
        this.redisProperties = redisProperties;  
    }  
}


```



##  spring cache


#### 依赖添加

```java
implementation 'org.springframework.boot:spring-boot-starter-cache'

```

#### yml添加

```yml

redisson:  
  cache:  
    ttl: 3600_000 #1小时过期时间  
    maxIdle: 600_000
    
```


#### service使用

```java
import lombok.extern.slf4j.Slf4j;  
import org.springframework.cache.annotation.Cacheable;  
import org.springframework.stereotype.Service;  
  
/**  
 * @author charles tao  
 */
@Service  
@Slf4j  
public class TestService {  
    
    @Cacheable(value = "testCache", key = "#id")  
    public String testCache(String id) {  
        log.error("no cache!!!!!");  
        return "res" + id;  
    } 
     
}
```

##  Redisson使用分布式锁

### 默认锁 （[非公平锁](锁.md#非公平锁) ）

效率高

```java
RLock lock = redissonClient.getLock("lock");  
try {  
    if (lock.tryLock(5, TimeUnit.SECONDS)) {  
        ///do something here...  
    }  
} catch (InterruptedException e) {  
    log.error("error:", e);  
    throw new InterruptedException("");  
} finally {  
    if (lock.isLocked()) {  
        lock.unlock();  
    }  
}
```

### 公平锁

```java

RLock lock = redissonClient.getFairLock(lockKey); // 获取公平锁

```

### 读写锁

读写锁的特点就是并发性能高，它是允许多个线程同时获取读锁进行读操作的，也就是说在没有写锁的情况下，读取操作可以并发执行，提高了系统的并行度。但写锁则是独占式的，同一时间只有一个线程可以获得写锁，无论是读还是写都无法与写锁并存，这样就确保了数据修改时的数据一致性。

```java

RReadWriteLock lock = redissonClient.getReadWriteLock(lockKey); // 获取读写锁 
lock.readLock(); // 读锁 
lock.writeLock(); // 写锁

```
### 联锁

Redisson 也支持联锁，也叫分布式多锁 MultiLock，它允许客户端一次性获取多个独立资源（RLock）上的锁，这些资源可能是不同的键或同一键的不同锁。当所有指定的锁都被成功获取后，才会认为整个操作成功锁定。这样能够确保在分布式环境下进行跨资源的并发控制。

```java

RLock lock1 = redisson.getLock("lock1");  
RLock lock2 = redisson.getLock("lock2");  
// 联锁  
RedissonMultiLock multiLock = new RedissonMultiLock(lock1, lock2);  
try {  
        // 一次性尝试获取所有锁  
        if (multiLock.tryLock()) {  
        // 获取锁成功...  
        }  
        } finally {  
        // 释放所有锁  
        multiLock.unlock();  
}

```

### redisonn redlock (增强联锁 已弃用)

==对集群的每个节点进行加锁，如果大多数（N/2+1）加锁成功了，则认为获取锁成功。==

#### 主从结构中锁的小概率锁失效为题

1. 客户端 A 从 master 获取到锁
2. 在 master 将锁同步到 slave 之前，master 宕掉了。
3. slave 节点被晋级为 master 节点
4. 客户端 B 取得了同一个资源被客户端 A 已经获取到的另外一个锁。**安全失效！**

#### redlock 实现

1. 客户端 A 锁住了集群的大多数（一半以上）；
2. 客户端 B 也要锁住大多数；
3. 这里肯定会冲突，所以 客户端 B 加锁失败。


## 分布式自增ID

ID 是数据的唯一标识，传统的做法是利用 UUID 和数据库的自增 ID。

但由于 UUID 是无序的，不能附带一些其他信息，因此实际作用有限。

随着业务的发展，数据量会越来越大，需要对数据进行分表，甚至分库。分表后每个表的数据会按自己的节奏来自增，这样会造成 ID 冲突，因此这时就需要一个单独的机制来负责生成唯一 ID，redis 原生支持生成全局唯一的 ID。

简单样例如下！

```java
final String lockKey = "aaaa";
//通过redis的自增获取序号
RAtomicLong atomicLong = redissonClient.getAtomicLong(lockKey);
//设置过期时间
atomicLong.expire(30, TimeUnit.SECONDS);
// 获取值
System.out.println(atomicLong.incrementAndGet());
```


## redisson 分布式延迟队列

花了一天研究了下Redisson 的延时队列，RBlockingQueue ，RDelayedQueue 。 网上没一个说清楚的，而且都是说**轮询redis的zset，都是错误的**！ 让我来纠正，如果我有错的也可指出。

### Demo用法

```java
public static void main(String[] args) throws InterruptedException, UnsupportedEncodingException {
	Config config = new Config();
	config.useSingleServer().setAddress("redis://172.29.2.10:7000");
	RedissonClient redisson = Redisson.create(config);
	RBlockingQueue<String> blockingQueue = redisson.getBlockingQueue("dest_queue1");
	RDelayedQueue<String> delayedQueue = redisson.getDelayedQueue(blockingQueue);
	new Thread() {
		public void run() {
			while(true) {
				try {
                                        //阻塞队列有数据就返回，否则wait
					System.err.println( blockingQueue.take());
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		};
	}.start();
	
	for(int i=1;i<=5;i++) {
                // 向阻塞队列放入数据
		delayedQueue.offer("fffffffff"+i, 13, TimeUnit.SECONDS);
	}
}
```

上面构造了Redisson 阻塞延时队列，然后向里面塞了5条数据，都是13秒后到期。**我们先不启动程序**，先打开redis执行：

```js
[root@localhost redis-cluster]# redis-cli -c -p 7000 -h 172.29.2.10 --raw
172.29.2.10:7000> monitor
OK
```

**monitor** 命令可以监控redis执行了哪些命令，**注意线上不要乱搞，耗性能的**。然后我们启动程序，观察redis执行命令情况，这里分为三个阶段：

### **第一介段： 客户端程序启动，offer方法执行之前 ，redis服务会收到如下redis命令：**

```js
1610452446.652126 [0 172.29.2.194:65025] "SUBSCRIBE" "redisson_delay_queue_channel:{dest_queue1}"
1610452446.672009 [0 lua] "zrangebyscore" "redisson_delay_queue_timeout:{dest_queue1}" "0" "1610452442403" "limit" "0" "2"
1610452446.672018 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0" "WITHSCORES"
1610452446.673896 [0 172.29.2.194:65034] "BLPOP" "dest_queue1" "0" 
```

**SUBSCRIBE**

这里订阅了一个固定的队列 redisson_delay_queue_channel:{dest_queue1}， 就是为了开启进程里面的延时任务，很重要，redisson延时取数据都靠它了。后面会说。

**zrangebyscore**

```abap
zrangebyscore用法扫盲
>> zrangebyscore key min max [WITHSCORES] [LIMIT offset count]
分页获取指定区间内(min - max)，带有分数值(可选)的有序集成员的列表。
```

redisson_delay_queue_timeout:{dest_queue1} 是一个zset，当有延时数据存入Redisson队列时，就会在此队列中插入 数据，排序分数为延时的时间戳。

zrangebyscore就是取出前2条（源码是100条，如下图）过了当前时间的数据。如果取的是0的话就执行下面的zrange， 这里程序刚启动肯定是0（除非是之前的队列数据没有取完）。之所以在刚启动时 这样取数据就是为了把上次进程宕机后没发完的数据发完。

![[redis/_resources/redisson/43a125fe84975cd2dee7fedfe8da2666_MD5.webp]]

**zrange**

取出第一个数，也就是判断上面的还有不有下一页。

**BLPOP**

移出并获取 dest_queue1 列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止 ， 这里显然没有元素 ，就会一直阻塞。

  

### 第二介段： 执行offer向Redisson队列设置值

这个阶段我们发现redis干了下面事情：

```java
1610452446.684465 [0 lua] "zadd" "redisson_delay_queue_timeout:{dest_queue1}" "1610452455407" ":\xdf\x0eO\x8c\xa7\xd4C\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff1"
1610452446.684480 [0 lua] "rpush" "redisson_delay_queue:{dest_queue1}" ":\xdf\x0eO\x8c\xa7\xd4C\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff1"
1610452446.684492 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0"
1610452446.684498 [0 lua] "publish" "redisson_delay_queue_channel:{dest_queue1}" "1610452455407"
1610452446.687922 [0 lua] "zadd" "redisson_delay_queue_timeout:{dest_queue1}" "1610452455422" "e\xfd\xfe?j?\xdbC\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff2"
1610452446.687943 [0 lua] "rpush" "redisson_delay_queue:{dest_queue1}" "e\xfd\xfe?j?\xdbC\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff2"
1610452446.687958 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0"
1610452446.690478 [0 lua] "zadd" "redisson_delay_queue_timeout:{dest_queue1}" "1610452455424" "\x80J\x01j\x11\xee\xda\xc3\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff3"
1610452446.690492 [0 lua] "rpush" "redisson_delay_queue:{dest_queue1}" "\x80J\x01j\x11\xee\xda\xc3\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff3"
1610452446.690502 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0"
1610452446.692661 [0 lua] "zadd" "redisson_delay_queue_timeout:{dest_queue1}" "1610452455427" "v\xb5\xd0r\xb48\xd4\xc3\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff4"
1610452446.692674 [0 lua] "rpush" "redisson_delay_queue:{dest_queue1}" "v\xb5\xd0r\xb48\xd4\xc3\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff4"
1610452446.692683 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0"
1610452446.696054 [0 lua] "zadd" "redisson_delay_queue_timeout:{dest_queue1}" "1610452455429" "\xe7\a\x8b\xee\t-\x94C\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff5"
1610452446.696081 [0 lua] "rpush" "redisson_delay_queue:{dest_queue1}" "\xe7\a\x8b\xee\t-\x94C\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff5"
1610452446.696098 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0"
```

我们客户端是设置了5条数据。上面也可以看出来。

**zadd**

往我们zset里面设置 数据截止的时间戳（当前执行的时间戳+延时的时间毫秒值），内容为我们的ffffff1 ，不过特殊编码了，加了点什么，不用管。

**rpush**

同步一份数据到list队列，这里也不知道干嘛的，先放到这里。

**zrange+publish**

取出排序好的第一个数据，也就是最临近要触发的数据，然后发送通知 （之前订阅了的客户端，可能是微服务就有多个客户端），内容为将要触发的时间。客户端收到通知后，就在自己进程里面开启延时任务（**HashedWheelTimer**），到时间后就可以从redis取数据发送。

后面又是我们的5条循环的设置数据 zadd...

  

### **第三介段：到延时时间取redis数据**

```java
1610452459.680953 [0 lua] "zrangebyscore" "redisson_delay_queue_timeout:{dest_queue1}" "0" "1610452455416" "limit" "0" "2"
1610452459.680967 [0 lua] "rpush" "dest_queue1" "\x04>\nfffffffff1"
1610452459.680976 [0 lua] "lrem" "redisson_delay_queue:{dest_queue1}" "1" ":\xdf\x0eO\x8c\xa7\xd4C\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff1"
1610452459.680984 [0 lua] "zrem" "redisson_delay_queue_timeout:{dest_queue1}" ":\xdf\x0eO\x8c\xa7\xd4C\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff1"
1610452459.680991 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0" "WITHSCORES" // 判断是否有值
1610452459.745813 [0 lua] "zrangebyscore" "redisson_delay_queue_timeout:{dest_queue1}" "0" "1610452455480" "limit" "0" "2"
1610452459.745829 [0 lua] "rpush" "dest_queue1" "\x04>\nfffffffff2"
1610452459.745837 [0 lua] "lrem" "redisson_delay_queue:{dest_queue1}" "1" "e\xfd\xfe?j?\xdbC\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff2"
1610452459.745845 [0 lua] "rpush" "dest_queue1" "\x04>\nfffffffff3"
1610452459.745848 [0 lua] "lrem" "redisson_delay_queue:{dest_queue1}" "1" "\x80J\x01j\x11\xee\xda\xc3\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff3"
1610452459.745855 [0 lua] "zrem" "redisson_delay_queue_timeout:{dest_queue1}" "e\xfd\xfe?j?\xdbC\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff2" "\x80J\x01j\x11\xee\xda\xc3\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff3"
1610452459.745864 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0" "WITHSCORES"
1610452459.756909 [0 172.29.2.194:65026] "BLPOP" "dest_queue1" "0"
1610452459.758092 [0 lua] "zrangebyscore" "redisson_delay_queue_timeout:{dest_queue1}" "0" "1610452455493" "limit" "0" "2"
1610452459.758108 [0 lua] "rpush" "dest_queue1" "\x04>\nfffffffff4"
1610452459.758114 [0 lua] "lrem" "redisson_delay_queue:{dest_queue1}" "1" "v\xb5\xd0r\xb48\xd4\xc3\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff4"
1610452459.758121 [0 lua] "rpush" "dest_queue1" "\x04>\nfffffffff5"
1610452459.758124 [0 lua] "lrem" "redisson_delay_queue:{dest_queue1}" "1" "\xe7\a\x8b\xee\t-\x94C\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff5"
1610452459.758133 [0 lua] "zrem" "redisson_delay_queue_timeout:{dest_queue1}" "v\xb5\xd0r\xb48\xd4\xc3\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff4" "\xe7\a\x8b\xee\t-\x94C\r\x00\x00\x00\x00\x00\x00\x00\x04>\nfffffffff5"
1610452459.758143 [0 lua] "zrange" "redisson_delay_queue_timeout:{dest_queue1}" "0" "0" "WITHSCORES"
1610452459.759030 [0 172.29.2.194:65037] "BLPOP" "dest_queue1" "0"
1610452459.760933 [0 172.29.2.194:65036] "BLPOP" "dest_queue1" "0"
1610452459.763913 [0 172.29.2.194:65038] "BLPOP" "dest_queue1" "0"
1610452459.765999 [0 172.29.2.194:65039] "BLPOP" "dest_queue1" "0"
```

这个阶段是由客户端进程里面的延时任务执行的，延时任务是在第二阶段构造的，已经说了（通过redis的订阅/发布实现）。

**zrangebyscore**

取出前2条到时间的数据，第一阶段已说。

**rpush**

将上面取到的数据push到阻塞队列，注意我们第一阶段已经监听了这个阻塞队列

```text
"BLPOP" "dest_queue1" "0" 
```

所以这里就会通知客户端取数据。

**lrem + zrem**

将取完的数据删掉。

**zrange**

取zset第一个数据，有的话继续上面逻辑取数据，否则进入下面。

**BLPOP**

继续监听这个阻塞队列。以便下次用。

### 小结一下

- 客户端启动，redisson先订阅一个key，同时 BLPOP key 0 无限监听一个阻塞队列（等里面有数据了就返回）。
- 当有数据put时，redisson先把数据放到一个zset集合（按延时到期时间的时间戳为分数排序），同时发布上面订阅的key，发布内容为距离当前最近一个要执行的数据到期的timeout，此时客户端进程开启一个延时任务，延时时间为发布的timeout。
- 客户端进程的延时任务到了时间执行，从zset分页取出过了当前时间的数据，然后将数据rpush到第一步的阻塞队列里。然后将当前数据从zset移除，取完之后，又执行 BLPOP key 0 无限监听一个阻塞队列。
- 上一步客户端监听的阻塞队列返回取到数据，回调到 RBlockingQueue 的 take方法。于是，我们就收到了数据。

**大致原理就是这样，redisson不是通过轮询zset的，将延时任务执行放到进程里面实现，只有到时间才会取redis zset。**

**redisson里面还有很多异常，重试机制 没讲。毕竟时间就一天，没法全部吃透。有了这些原理，我相信你也能实现一个属于自己的redisson延时队列了。**

### HashedWheelTimer 实现原理

HashedWheelTimer本质是一种类似延迟任务队列的实现，适用于对时效性不高的，可快速执行的，大量这样的“小”任务，能够做到高性能，低消耗

redisson是在这里用的 [org](eclipse-javadoc:%E2%98%82=redisson/src%5C/main%5C/java=/optional=/true=/=/maven.pomderived=/true=/%3Corg).[redisson](eclipse-javadoc:%E2%98%82=redisson/src%5C/main%5C/java=/optional=/true=/=/maven.pomderived=/true=/%3Corg.redisson).[connection](eclipse-javadoc:%E2%98%82=redisson/src%5C/main%5C/java=/optional=/true=/=/maven.pomderived=/true=/%3Corg.redisson.connection).MasterSlaveConnectionManager

```java
   // 初始化  timer 
   protected void initTimer(MasterSlaveServersConfig config) {
        int[] timeouts = new int[]{config.getRetryInterval(), config.getTimeout()};
        Arrays.sort(timeouts);
        int minTimeout = timeouts[0];
        if (minTimeout % 100 != 0) {
            minTimeout = (minTimeout % 100) / 2;
        } else if (minTimeout == 100) {
            minTimeout = 50;
        } else {
            minTimeout = 100;
        }
        // minTimeout 为100
        timer = new HashedWheelTimer(new DefaultThreadFactory("redisson-timer"), minTimeout, TimeUnit.MILLISECONDS, 1024, false);
        
        connectionWatcher = new IdleConnectionWatcher(this, config);
        subscribeService = new PublishSubscribeService(this, config);
    }


    @Override
    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
        try {
        	System.err.println(time + " HadLuo ======================newTimeout==================" + task + " , "+ delay+"ms");
            return timer.newTimeout(task, delay, unit);
        } catch (IllegalStateException e) {
            if (isShuttingDown()) {
                return DUMMY_TIMEOUT;
            }
            
            throw e;
        }
    }
```

抽象出来就是这样

```java
HashedWheelTimer timer = new HashedWheelTimer(new DefaultThreadFactory("redisson-timer"), 100,
		TimeUnit.MILLISECONDS, 1024, false);
// 构建一个延时任务
timer.newTimeout((time) -> {
	System.err.println("到了12s后了，该娶媳妇了~");
}, 12, TimeUnit.SECONDS);
```


## [Redis commands mapping ](https://github.com/redisson/redisson/wiki/11.-Redis-commands-mapping)

