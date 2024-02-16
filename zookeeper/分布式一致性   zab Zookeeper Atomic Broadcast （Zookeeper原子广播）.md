### zab =》 paxos 协议的 简易实现


Zab 是特别为 Zookeeper 设计的支持崩溃恢复的原子广播协议，在 Zookeeper 中主要依赖 Zab 协议实现数据一致性，基于该协议，Zookeeper 实现了一种主备模型（Leader 与 Follower）的系统架构保证集群中各个副本之间的数据一致性。

### Zab 协议核心

在 Zookeeper 中只有一个 Leader，并且只有 Leader 可以处理外部客户端的事务请求，并将其转换成一个事务 Proposal（写操作），然后 Leader 服务器再将事务 Proposal 操作的数据同步到所有 Follower（数据广播/数据复制）。

Zookeeper 采用 Zab 协议的核心就是只要有一台服务器提交了 Proposal，就要确保所有服务器最终都能正确提交 Proposal，这也是 CAP/BASE 最终实现一致性的体现。

### Zab 模式

Zab 协议有两种模式：一种是消息广播模式，另一种是崩溃恢复模式。

#### 消息广播模式

在 Zookeeper 集群中数据副本的传递策略就是采用消息广播模式，Zookeeper 中的数据副本同步方式与2PC方式相似但却不同，2PC是要求协调者必须等待所有参与者全部反馈ACK确认消息后，再发送 commit 消息，要求所有参与者要么全成功要么全失败，2PC方式会产生严重的阻塞问题。

而 Zookeeper 中 Leader 等待 Follower 的 ACK 反馈是指：只要半数以上的 Follower 成功反馈即可，不需要收到全部的 Follower 反馈。

![[zookeeper/_resources/分布式一致性   zab Zookeeper Atomic Broadcast （Zookeeper原子广播）/0bdd7391273f1916d2fd8e68de61f15c_MD5.webp]]


Zookeeper 中广播消息步骤：

- 客户端发起一个写操作请求
- Leader 服务器处理客户端请求后将请求转换为 Proposal，同时为每个 Proposal 分配一个全局唯一 ID，即 ZXID
- Leader 服务器与每个 Follower 之间都有一个队列，Leader 将消息发送到该队列
- Follower 机器从队列中取出消息处理完（写入本地事务日志中）后，向 Leader 服务器发送 ACK 确认
- Leader 服务器收到半数以上的 Follower 的 ACK 后，即认为可以发送 Commit
- Leader 向所有的 Follower 服务器发送 Commit 消息

#### 崩溃恢复模式

**一旦 Leader 服务器出现崩溃或者由于网络原因导致 Leader 服务器失去了与过半 Follower 的联系，那么就会进入崩溃恢复模式。**

Zookeeper 集群中为保证任何进程能够顺序执行，只能是 Leader 服务器接收写请求，其他服务器接收到客户端的写请求，也会转发至 Leader 服务器进行处理。

**Zab 协议崩溃恢复需满足以下2个请求：**

- 确保已经被 Leader 提交的 proposal 必须最终被所有的 Follower 服务器提交
- 确保丢弃已经被 Leader 提出的但没有被提交的 Proposal

也就是新选举出来的 Leader 不能包含未提交的 Proposal，必须都是已经提交了的 Proposal 的 Follower 服务器节点，新选举出来的 Leader 节点中含有最高的 ZXID，所以，在 Leader 选举时，将 ZXID 作为每个 Follower 投票时的信息依据。这样做的好处是避免了 Leader 服务器检查 Proposal 的提交和丢弃工作。

### Leader 选举算法

可以通过配置项设置 Zookeeper 选举 Leader 的算法，可选项有：

- 0：基于UDP的 LeaderElection
- 1：基于UDP的 FastLeaderElection
- 2：基于UDP和认证的 FastLeaderElection
- 3：基于TCP的FastLeaderElection

在 3.4.10 版本中，默认值为3，另外三种算法已被弃用。下面重点介绍 基于TCP的FastLeaderElection

#### FastLeaderElection 原理

##### myid

每个 Zookeeper 服务器，都需要在数据文件夹下创建一个名为 myid 的文件，该文件包含整个 Zookeeper 集群唯一的 ID，例如，某个 Zookeeper 集群包含三台服务器，hostname 分别为 zoo1,zoo2,zoo3，其中 myid 分别为1,2,3,则在配置文件中其 ID 与 hostname 必须一一对应，如在配置文件中，server.后面的数据即为 myid


```text
server.1=zoo1:2888:3888 
server.2=zoo2:2888:3888 
server.3=zoo3:2888:3888
```

##### ZXID

类似于 RDBMS 中的事务ID，用于标识一个 Proposal ID，为了保证顺序性，ZXID 必须单调递增，因此 Zookeeper 使用一个 64 位的数来表示，高 32 位是 Leader 的 epoch，从 1 开始，每次选出新的 Leader，epoch 加 1，低 32 位为该 epoch 内的序号，每次 epoch 变化，都将低 32 位的序号重置，这样保证了 ZXID 的全局递增性。

##### 服务器状态

- Looking：不确定Leader状态，该状态下的服务器认为当前集群中没有Leader，会发起Leader选举
- Following：跟随者状态，表明当前服务器角色是Follower，并且它知道Leader是谁
- Leading：领导者状态，表明当前服务器角色是Leader，它会维护与Follower间的心跳
- Observing：观察者状态，表明当前服务器角色是Observer，与Follower唯一的不同在于不参与选举，也不参与集群写操作时的投票

##### 选票数据结构

每个服务器在进行领导选举时，会发送如下关键信息：

- **logicClock** 每个服务器会维护一个自增的整数，名为logicClock，它表示这是该服务器发起的第多少轮投票
- **state** 当前服务器的状态
- **self_id** 当前服务器的myid
- **self_zxid** 当前服务器上所保存的数据的最大zxid
- **vote_id** 被推举的服务器的myid
- **vote_zxid** 被推举的服务器上所保存的数据的最大zxid

##### 自增选举轮次

Zookeeper 规定所有有效的投票都必须在同一轮次中，每个服务器在开始新一轮投票时，会先对自己维护的 logicClock 进行自增操作。

**初始化选票**：

每个服务器在广播自己的选票前会将自己的投票箱清空，该投票箱记录了所收到的选票。

**发送初始化选票**

每个服务器最开始都是通过广播把票投给自己

**接收外部投票**

服务器会尝试从其他服务器获取投票，并计入自己的投票箱内，如果无法获取任何外部投票，则会确认自己是否与集群中其他服务器保持着有效连接，如果是，则发送自己的投票，如果否，则马上与之建立连接。

**判断选举轮次**

收到外部投票后，首先会根据投票信息中所包含的 logicClock 来进行不同处理：

- 外部投票的 logicClock 大于自己的 logicClock，说明该服务器的选举轮次落后于其他服务器轮次，立即清空自己的投票箱并将自己的 logicClock 更新为收到的 logicClock，然后再对自己之前的投票与收到的投票以确定是否需要变更自己的投票，最终再次将自己的投票广播出去。
- 外部投票的logicClock小于自己的logicClock。当前服务器直接忽略该投票，继续处理下一个投票
- 外部投票的logickClock与自己的相等，当时进行选票PK

**选票PK**

基于(self_id, self_zxid)与(vote_id, vote_zxid)的对比：

- 外部投票的logicClock大于自己的logicClock，则将自己的logicClock及自己的选票的logicClock变更为收到的logicClock
- 若logicClock一致，则对比二者的vote_zxid，若外部投票的vote_zxid比较大，则将自己的票中的vote_zxid与vote_myid更新为收到的票中的vote_zxid与vote_myid并广播出去，另外将收到的票及自己更新后的票放入自己的票箱。如果票箱内已存在(self_myid, self_zxid)相同的选票，则直接覆盖
- 若二者vote_zxid一致，则比较二者的vote_myid，若外部投票的vote_myid比较大，则将自己的票中的vote_myid更新为收到的票中的vote_myid并广播出去，另外将收到的票及自己更新后的票放入自己的票箱

**统计选票**

如果已经确定有过半服务器认可了自己的投票（可能是更新后的投票），则终止投票。否则继续接收其它服务器的投票。

**更新服务器状态**

投票终止后，服务器开始更新自身状态。若过半的票投给了自己，则将自己的服务器状态更新为LEADING，否则将自己的状态更新为FOLLOWING。
