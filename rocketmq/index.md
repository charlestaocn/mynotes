这个indexFile中的索引数据是包含Key的消息被发送到Broker时写入的。如果消息中没有Key，不会被写入。

Message Key 一般用于消息在业务层面的唯一标识。对发送的消息设置好 Key，以后可以根据这个 Key 来查找消息。比如消息异常，消息丢失，进行查找会很 方便。RocketMQ 会创建专门的索引文件，用来存储 Key 与消息的映射，由于是 Hash 索引，应尽量使 Key 唯一，避免潜在的哈希冲突

RocketMQ 并不能保证 message id 唯一，在这种情况下，生产者在 push 消息的时候可以给每条消息设定唯一的 key, 消费者可以通过 message key 保证对消息幂等处理。

  

## **RocketMQ的Index File**

Index File由三部分组成：Index Head、Hash Slot、Index Item

  

![[rocketmq/_resources/index/3675353ae17437b9ac29689704a97647_MD5.png]]

  

## **Index Item的结构

  

![[rocketmq/_resources/index/7e14532c5b31670c89a56eb86c9dd702_MD5.png]]

  

**看完这两个没概念？没事，有个印象就好，重点看下面索引的生成**

## **索引的生成

1. 插入一条消息
2. 计算出消息key的hashcode
3. 根据hashcode%500w，计算出应该放到哪个槽中
4. 然后在插入Index Item，并在槽中记录Index Item的位置

**假设hash slot只有5个、index item 只有20个，演示一下插入索引的过程**

  

![[rocketmq/_resources/index/1b7e239096105a6457338df5fc134a98_MD5.webp]]

  

插入key1, 计算出hashcode=5，5%5=0在hashslot的0位置

在index item list中插入一个Index Item

hashslot 0位置指向插入的Index item

  

![[rocketmq/_resources/index/1339982b48ee9c74818e00d49c68ff23_MD5.webp]]

  

插入key2, 计算出hashcode=7，7%5=2在hashslot的0位置

在index item list中插入一个Index Item

hashslot 2位置指向插入的Index item

  

![[rocketmq/_resources/index/068dbf9b3c00fed989b2a76367a2419f_MD5.webp]]

  

插入key3, 计算出hashcode=10，10%5=0在hashslot的0位置

此时发生了hash碰撞，它在slot的位置和key1相同，怎么处理？

在index item list中插入一个Index Item

hashslot 0位置指向插入的Index item key3

  

![[rocketmq/_resources/index/ccd599ad1fa2ae417e82460a444427ae_MD5.webp]]

  

那key1岂不是变成一个孤岛了？

Index item key3会将它的preIndexNo指向Index item key1，这样index item key3和index item key1形成了一个单向链表

  

![[rocketmq/_resources/index/34aa661cb16f7e84a048a039460f034f_MD5.png]]

  

## **索引查找

假设要查询的key是key1

1. 根据key计算出所在hash槽，计算出位置是0
2. 根据hash槽的值定位对应的Index item, 定位到key3的index item，它是一个单向链表，key3 -> key1
3. 遍历这个链表，定位到key1
4. 根据item的pyhoffset，到commitLog中定位到消息

## **Index Head

- **beginTimestamp**：文件中消息的最小存储时间
- **endTimestamp**：文件中消息的最大存储时间
- **beginPhyoffset**：消息的最小偏移量
- **endphyoffset**：消息的最大物理偏移量
- **hashSlotCount**：已用 hash 槽个数
- **indexCount**：已用 index 个数