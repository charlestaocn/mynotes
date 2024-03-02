
## ==(部分属性被省略)==

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201104070257582.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p3eDkwMDEwMg==,size_16,color_FFFFFF,t_70#pic_center)




#### rehash操作

ht[2]定义了两个哈希表，ht[0]和ht[1]。而Redis在默认情况下使用的是ht[0]，不会为ht[1]初始化分配空间。

当设置一个哈希对象时，具体会落到哈希数组(上图中的dictEntry*[3])中的哪个下标，是通过计算哈希值来确定的，如果发生哈希碰撞，那么同一个下标就会有多个dictEntry，从而形成一个链表(最后插入的总是落在链表的最前面)，链表越长，性能越差。所以为了保证哈希表的性能，需要在满足以下两个条件中的一个时，对哈希表进行rehash(重新散列)操作：

> 1. 负载因子大于等于1且dict_can_resize设置为1时
> 2. 负载因子大于等于安全阈值(dict_force_resize_ratio=5)时
  PS：负载因子=哈希表已使用节点数/哈希表大小(即：h[0].used/h[0].size)。


#### rehash步骤

扩展哈希和收缩哈希都是通过执行rehash来完成，主要经过以下五步：

> 1. 为字典dict的ht[1]哈希表分配空间，其大小为当前哈希表已保存节点数(即：ht[0].used)的 两倍
>2. 将字典中的属性rehashix的值设置为0，表示正在执行rehash操作。
  3. 将ht[0]中所有的键值对依次重新计算哈希值，并放到ht[1]数组对应位置，完成一个键值对的rehash之后rehashix的值需要加1。
>4. 当ht[0]中所有的键值对都迁移到ht[1]之后，释放ht[0]，并将ht[1]修改为ht[0]，然后再创建一个新的ht[1]数组，为下一次rehash做准备。
>5. 将字典中的属性rehashix设置为-1，表示rehash已经结束

#### 渐进式rehash

> 每次执行增删改查命令时会带一链老的数据到新的

上面介绍的这种方式因为不是一次性全部rehash，而是分多次来慢慢的将ht[0]中的键值对rehash到ht[1]的操作就称之为渐进式rehash。渐进式rehash可以避免了集中式rehash带来的庞大计算量，采用了分而治之的思想。

在渐进式rehash过程中，因为还可能会有新的键值对存进来，此时Redis的做法是新添加的键值对统一放入ht[1]中，这样就确保了ht[0]键值对的数量只会减少。

当执行rehash操作时需要执行查询操作，此时会先查询ht[0]，查找不到结果再到ht[1]中查询。



### ziplist和hashtable的编码转换

当一个哈希对象可以满足以下两个条件时，哈希对象会选择使用ziplist编码来进行存储：

1、哈希对象中的所有键值对总长度(包括键和值)小于64字节（这个阈值可以通过参数hash-max-ziplist-value 来进行控制）。
2、哈希对象中的键值对数量小于512个（这个阈值可以通过参数hash-max-ziplist-entries 来进行控制）。
一旦不满足这两个条件中的任意一个，哈希对象就会选择使用hashtable来存储。

