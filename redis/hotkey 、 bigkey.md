## hotkey 热点key

所谓热key问题就是，突然有几十万的请求去访问redis上的某个特定key。那么，这样会造成流量过于集中，==达到物理网卡上限==，从而导致这台redis的服务器宕机。

由于某个 `Key` 的数据一定是存储到后端某台服务器的 `Redis` 单个实例上，如果对这个 `Key` 突然出现大量的请求操作，这样就会造成流量过于集中，达到 `Redis` 单个实例处理上限，可能会导致 `Redis` 实例 `CPU` 使用率 `100%`，或者是网卡流量达到上限等，对系统的稳定性和可用性造成影响，或者更为严重出现服务器宕机，无法对外提供服务；更有甚者在出现 `Redis` 服务不可用之后，大量的数据请求全部落地数据库查询上，`Redis` 都已经顶不住了，数据库也是分分钟挂掉的节奏。

- 流量集中，达到服务器处理上限（`CPU`、网络 `IO` 等）；
- 会影响在同一个 `Redis` 实例上其他 `Key` 的读写请求操作；
- 热 `Key` 请求落到同一个 `Redis` 实例上，无法通过扩容解决；
- 大量 `Redis` 请求失败，查询操作可能打到数据库，拖垮数据库，导致整个服务不可用。


#### 如何发现热 Key

##### 凭借业务经验，预估热 Key 出现

根据业务系统上线的一些活动和功能，我们是可以在某些场景下提前预估热 `Key` 的出现的，比如业务需要进行一场商品秒杀活动，秒杀商品信息和数量一般都会缓存到 `Redis` 中，这种场景极有可能出现热 `Key` 问题的。

- 优点：简单，凭经验发现热 `Key`，提早发现提早处理；
- 缺点：没有办法预测所有热 `Key` 出现，比如某些热点新闻事件，无法提前预测。

##### 客户端进行收集

一般我们在连接 `Redis` 服务器时都要使用专门的 SDK（比如：`Java` 的客户端工具 `Jedis`、`Redisson`），我们可以对客户端工具进行封装，在发送请求前进行收集采集，同时定时把收集到的数据上报到统一的服务进行聚合计算。

- 优点：方案简单
- 缺点：
- 对客户端代码有一定入侵，或者需要对 `SDK` 工具进行二次开发；
- 没法适应多语言架构，每一种语言的 `SDK` 都需要进行开发，后期开发维护成本较高。

##### 在代理层进行收集

如果所有的 `Redis` 请求都经过 `Proxy`（代理）的话，可以考虑改动 `Proxy` 代码进行收集，思路与客户端基本类似。

  

![[redis/_resources/redis  hotkey 、 bigkey/2c8e1e9b1a3374f6894ae70a71b03d2e_MD5.webp]]

  

**hotkeys 参数**

`Redis` 在 `4.0.3` 版本中添加了 [hotkeys](https://link.zhihu.com/?target=https%3A//github.com/redis/redis/pull/4392) 查找特性，可以直接利用 `redis-cli --hotkeys` 获取当前 `keyspace` 的热点 `key`，实现上是通过 `scan + object freq` 完成的。

**[monitor](https://link.zhihu.com/?target=https%3A//redis.io/commands/monitor) 命令**

`monitor` 命令可以实时抓取出 `Redis` 服务器接收到的命令，通过 `redis-cli monitor` 抓取数据，同时结合一些现成的分析工具，比如 [redis-faina](https://link.zhihu.com/?target=https%3A//github.com/facebookarchive/redis-faina)，统计出热 Key。

##### Redis 节点抓包分析

`Redis` 客户端使用 `TCP` 协议与服务端进行交互，通信协议采用的是 `RESP` 协议。自己写程序监听端口，按照 `RESP` 协议规则解析数据，进行分析。或者我们可以使用一些抓包工具，比如 `tcpdump` 工具，抓取一段时间内的流量进行解析。

### 热 Key 问题解决方案

##### 增加 Redis 实例复本数量

对于出现热 `Key` 的 `Redis` 实例，我们可以通过水平扩容增加副本数量，将读请求的压力分担到不同副本节点上。

##### 二级缓存（本地缓存）

当出现热 `Key` 以后，把热 `Key` 加载到系统的 `JVM` 中。后续针对这些热 `Key` 的请求，会直接从 `JVM` 中获取，而不会走到 `Redis` 层。这些本地缓存的工具很多，比如 `Ehcache`，或者 `Google Guava` 中 `Cache` 工具，或者直接使用 `HashMap` 作为本地缓存工具都是可以的。

**使用本地缓存需要注意两个问题：**

- 如果对热 `Key` 进行本地缓存，需要防止本地缓存过大，影响系统性能；
- 需要处理本地缓存和 `Redis` 集群数据的一致性问题。

##### 热 Key 备份

通过前面的分析，我们可以了解到，之所以出现热 `Key`，是因为有大量的对同一个 `Key` 的请求落到同一个 `Redis` 实例上，如果我们可以有办法将这些请求打散到不同的实例上，防止出现流量倾斜的情况，那么热 `Key` 问题也就不存在了。

那么如何将对某个热 `Key` 的请求打散到不同实例上呢？我们就可以通过热 `Key` 备份的方式，基本的思路就是，我们可以给热 `Key` 加上前缀或者后缀，把一个热 `Key` 的数量变成 `Redis` 实例个数 `N` 的倍数 `M`，从而由访问一个 `Redis` `Key` 变成访问 `N * M` 个 `Redis` `Key`。 `N * M` 个 `Redis` `Key` 经过分片分布到不同的实例上，将访问量均摊到所有实例。

  

![[redis/_resources/redis  hotkey 、 bigkey/41cb6255715330844bb89a32b7269379_MD5.webp]]

  

在这段代码中，通过一个大于等于 `1` 小于 `M` 的随机数，得到一个 `bakHotKey`，程序会优先访问 `bakHotKey`，在得不到数据的情况下，再访问原来的 `hotkey`，并将 `hotkey` 的内容写回 `bakHotKey`。值得注意的是，`bakHotKey` 的过期时间是 `hotkey` 的过期时间加上一个较小的随机正整数，这是通过坡度过期的方式，保证在 `hotkey` 过期时，所有 `bakHotKey` 不会同时过期而造成缓存雪崩。

使用京东框架发现热key，[https://gitee.com/jd-platform-opensource/hotkey](https://link.zhihu.com/?target=https%3A//gitee.com/jd-platform-opensource/hotkey)
### 总结

在这一篇文章中我们首先分析了在 `Redis` 中热 `Key` 带来的一些问题，同时也介绍了在海量的 `Redis` `Key` 中找到热 `Key` 的一些方法，最后也提到了在解决热 `Key` 问题中我们常用的一些办法；总结来说，`Redis` 热 `Key` 问题首先是请求流量过大造成的，但是更深层次原因还是出现了流量倾斜，单个 `Redis` 实例承担的流量过大造成的，了解到了本质原因，解决的思路也就简单了，就是要想尽一切办法将单个实例承担的流量打散，让每个机器均衡承担热 `Key` 的流量，不要出现流量倾斜，保证系统的稳定性。

## bigkeys

通常我们会将含有较大数据或含有大量成员、列表数的Key称之为大Key，下面我们将用几个实际的例子对大Key的特征进行描述：

- 一个STRING类型的Key，它的值为5MB（数据过大）
- 一个LIST类型的Key，它的列表数量为20000个（列表数量过多）
- 一个ZSET类型的Key，它的成员数量为10000个（成员数量过多）
- 一个HASH格式的Key，它的成员数量虽然只有1000个但这些成员的value总大小为100MB（成员体积过大）


#### 查看bigkeys

```redis
./redis-cli --bigkeys


# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest list   found so far '"testList"' with 5 items
[00.00%] Biggest hash   found so far '"{cacheMap}:redisson_options"' with 2 fields
[00.00%] Biggest string found so far '"testBK"' with 6 bytes
[00.00%] Biggest set    found so far '"testSet"' with 2 members

-------- summary -------

Sampled 5 keys in the keyspace!
Total key length in bytes is 57 (avg len 11.40)

Biggest   list found '"testList"' has 5 items
Biggest   hash found '"{cacheMap}:redisson_options"' has 2 fields
Biggest string found '"testBK"' has 6 bytes
Biggest    set found '"testSet"' has 2 members

1 lists with 5 items (20.00% of keys, avg size 5.00)
2 hashs with 3 fields (40.00% of keys, avg size 1.50)
1 strings with 6 bytes (20.00% of keys, avg size 6.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
1 sets with 2 members (20.00% of keys, avg size 2.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)


```


#### **大Key带来的常见问题**

- Client发现Redis变慢；
- Redis内存不断变大引发OOM，或达到maxmemory设置值引发写阻塞或重要Key被逐出；
- Redis Cluster中的某个node内存远超其余node，但因Redis Cluster的数据迁移最小粒度为Key而无法将node上的内存均衡化；
- 大Key上的读请求使Redis占用服务器全部带宽，自身变慢的同时影响到该服务器上的其它服务；
- 删除一个大Key造成主库较长时间的阻塞并引发同步中断或主从切换；
#### 解决

**大Key的常见处理办法**

**对大Key进行拆分**

如将一个含有数万成员的HASH Key拆分为多个HASH Key，并确保每个Key的成员数量在合理范围，在Redis Cluster结构中，大Key的拆分对node间的内存平衡能够起到显著作用。

**对大Key进行清理**

将不适合Redis能力的数据存放至其它存储，并在Redis中删除此类数据。需要注意的是，我们已在上文提到一个过大的Key可能引发Redis集群同步的中断，Redis自4.0起提供了UNLINK命令，该命令能够以非阻塞的方式缓慢逐步的清理传入的Key，通过UNLINK，你可以安全的删除大Key甚至特大Key。

**时刻监控Redis的内存水位**

突然出现的大Key问题会让我们措手不及，因此，在大Key产生问题前发现它并进行处理是保持服务稳定的重要手段。我们可以通过监控系统并设置合理的Redis内存报警阈值来提醒我们此时可能有大Key正在产生，如：Redis内存使用率超过70%，Redis内存1小时内增长率超过20%等。

通过此类监控手段我们可以在问题发生前解决问题，如：LIST的消费程序故障造成对应Key的列表数量持续增长，将告警转变为预警从而避免故障的发生。

**对失效数据进行定期清理**

例如我们会在HASH结构中以增量的形式不断写入大量数据而忽略了这些数据的时效性，这些大量堆积的失效数据会造成大Key的产生，可以通过定时任务的方式对失效数据进行清理。在此类场景中，建议使用HSCAN并配合HDEL对失效数据进行清理，这种方式能够在不阻塞的前提下清理无效数据。