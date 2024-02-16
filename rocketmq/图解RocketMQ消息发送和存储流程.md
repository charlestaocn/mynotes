## 整体架构

![[rocketmq/_resources/图解RocketMQ消息发送和存储流程/887e688270b6d0b9bdf8fc38bde23082_MD5.png]]

- Producer：生产者
- Consumer：消费者
- Broker：负责消息存储、投递、查询
- NameServer：路由[注册中心](https://cloud.tencent.com/product/tse?from_column=20065&from=20065)。功能包括：Broker管理、路由信息管理

## 模块间数据流转

![[rocketmq/_resources/图解RocketMQ消息发送和存储流程/93653298c52ebb44eaf0edc73ccf1504_MD5.png]]

## 生产-消费模型

![[rocketmq/_resources/图解RocketMQ消息发送和存储流程/d1dadc687a6949c6d471cce0df838180_MD5.png]]

## 消息发送流程

![[rocketmq/_resources/图解RocketMQ消息发送和存储流程/8424c4371dabbf9e59b7a07c810dc711_MD5.png]]

- Broker启动时，向NameServer注册信息
- 客户端调用producer发送消息时，会先从NameServer获取该topic的路由信息。消息头code为GET_ROUTEINFO_BY_TOPIC
- 从NameServer返回的路由信息，包括topic包含的队列列表和broker列表
- Producer端根据查询策略，选出其中一个队列，用于后续存储消息
- 每条消息会生成一个唯一id，添加到消息的属性中。属性的key为UNIQ_KEY
- 对消息做一些特殊处理，比如：超过4M会对消息进行压缩
- producer向Broker发送rpc请求，将消息保存到broker端。消息头的code为SEND_MESSAGE或SEND_MESSAGE_V2（配置文件设置了特殊标志）

## 消息存储流程

![[rocketmq/_resources/图解RocketMQ消息发送和存储流程/4c3da849d696ca6162c6b5e731db6143_MD5.png]]

- Broker端收到消息后，将消息原始信息保存在CommitLog文件对应的MappedFile中，然后异步刷新到磁盘
- ReputMessageServie线程异步的将CommitLog中MappedFile中的消息保存到ConsumerQueue和IndexFile中
- ConsumerQueue和IndexFile只是原始文件的索引信息

### 消息体结构

![[rocketmq/_resources/图解RocketMQ消息发送和存储流程/9ce1cade56ebe730c1def813daf62856_MD5.png]]

- CommitLog的消息体长度不一样，每个CommitLog文件默认1G
- ConsumerQueue内的消息体长度固定，为20Byte

## 内存映射流程

![[rocketmq/_resources/图解RocketMQ消息发送和存储流程/2a8f901981ca17862225da064d4235c5_MD5.png]]

- 内存映射文件MappedFile通过AllocateMappedFileService创建
- MappedFile的创建是典型的生产者-消费者模型
- MappedFileQueue调用getLastMappedFile获取MappedFile时，将请求放入队列中
- AllocateMappedFileService线程持续监听队列，队列有请求时，创建出MappedFile对象
- 最后将MappedFile对象预热，底层调用force方法和mlock方法

## 刷盘流程

![[rocketmq/_resources/图解RocketMQ消息发送和存储流程/7886f59ce0fcbac64f02b554e48a2a53_MD5.png]]

- producer发送给broker的消息保存在MappedFile中，然后通过刷盘机制同步到磁盘中
- 刷盘分为同步刷盘和异步刷盘
- 异步刷盘后台线程按一定时间间隔执行
- 同步刷盘也是生产者-消费者模型。broker保存消息到MappedFile后，创建GroupCommitRequest请求放入列表，并阻塞等待。后台线程从列表中获取请求并刷新磁盘，成功刷盘后通知等待线程。

## 消费者组

Topic是相当于一种消息类型，而队列queue则是属于某个Topic下的更细分的一种单元。举个例子。Topic代表老虎，是一种动物类型，而队列就相当于东北虎，是对老虎的更详细描述。

在同一个消费者组下的消费者，不能同时消费同一个queue。

一个消费者组下的消费者，可以同时消费同一个Topic下的不同队列的消息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2911b9bca6814f15ac4a8fe2358f15ee.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3l1YW5jaGFuZ2xpYW5n,size_16,color_FFFFFF,t_70)
不同消费者组下的消费者，可以同时消费同一个Topic下的相同队列的消息

![在这里插入图片描述](https://img-blog.csdnimg.cn/1f8a917c2f6640f391e1604a409a97cd.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3l1YW5jaGFuZ2xpYW5n,size_16,color_FFFFFF,t_70)

同消费者组下的消费者，不可以同时消费不同Topic下的消息

![在这里插入图片描述](https://img-blog.csdnimg.cn/86023764b9e348d7ae5f881b46a4a221.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3l1YW5jaGFuZ2xpYW5n,size_16,color_FFFFFF,t_70)
