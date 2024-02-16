## 前言 

  在 Redis 的 列表(list) 命令中，有一些命令是阻塞模式的，比如：BRPOP,  BLPOP, BRPOPLPUSH, 这些命令都有可能造成客户端的阻塞。下面总结一下 Redis 实现阻塞和取消阻塞的过程。

## 阻塞过程

  当一个阻塞原语的处理目标为空键时， 执行该阻塞原语的客户端就会被阻塞。有以下步骤：

1：将客户端的状态设为“正在阻塞”， 并记录阻塞这个客户端的各个键，以及阻塞的最长时限(timeout) 等数据；

2：将客户端的信息记录到 server.db[i]->blocking_keys 中(其中 i 为客户端所使用的数据库号码)；

3：继续维持客户端和服务器之间的网络连接，但不再向客户端传送任何信息，造成客户端阻塞；

note: step2 中 service.db[i]->blocking_keys 是一个字典，键是那些造成客户端阻塞的键， 值是一个链表，链表里保存了所有因这个键而被阻塞的客户端，如下图所示：

![[redis/_resources/list 阻塞命令/acdf869cb6d9a71f3968ce852ad3b5c4_MD5.png]]


##  阻塞的取消过程

  阻塞的取消有三种方法：

    【1】被动脱离：有其它客户端为造成阻塞的键推入了新元素；

    【2】主动脱离：到达执行阻塞原语时设定的最大阻塞时间(timeout)；

    【3】强制脱离：客户端强制终止和服务端的连接，或者服务器停机；

### 被动脱离

    阻塞因 LPUSH, RPUSH, LINSERT 等添加命令而被取消，这三个添加新元素的命令，在底层都有一个 pushGenericCommand 的函数实现(在下方源码部分增加的 TODO 标志标识关键步骤）：


![[redis/_resources/list 阻塞命令/0557f2ca9bd5192fecb92c620c237199_MD5.png]]

pushGenericCommand 函数主要做了两件事情：

【1】向 readyList 添加到服务器；

【2】将新元素 value 添加到该 key 中

到此处为止，被该 key 阻塞的客户端还没有任何一个被解除阻塞状态，为了做到这一点，Redis 的主进程在执行完 pushGenericCommand 函数后，会继续调用 handleClientsBlockedOnLists  函数，


handleClientsBlockedOnLists  函数主要执行如下操作：

【1】如果 service.ready_keys 不为空，那么弹出该链表的表头元素，并取出其中的 readyList 值；

【2】根据 readyList 值中保存的 key 和 db, 在 service.blocking_keys 中查找所有因为该 key 而被阻塞的客户端(以链表形式保存)；

【3】如果 Key 不为空，那么从 Key 的列表中弹出一个元素，并获取客户端链表的第一个客户端，然后将被弹出元素作为被阻塞的客户端的返回值；

【4】根据 readyList 结构的属性，删除 service.blocking_keys 中相应的客户端数据，取消客户端的阻塞状态；

【5】继续执行步骤 【3】和 【4】，知道 key 没有元素可弹出，或者因为 key 而阻塞的客户端都取消阻塞为止；

【6】继续执行步骤 【1】，直到 ready_keys 字典所有链表里的所有 readyList 结构都被处理完为止；

### 主动脱离

    阻塞因超过最大等待时间而被取消。当客户端被阻塞时，所有造成它阻塞的键，以及阻塞的最长时限都会被记录在客户端里面，并且盖客户端的状态会被设置为”正在阻塞“。每次 Redis 服务器常规操作函数(redis.c/serverCron) 执行时，程序都会检查所有连接到服务器的客户端，查看哪些处于”正在阻塞“状态的客户端时限是否已经过期，如果是的话，就给客户端返回一个空白回复，然后撤销对客户端的阻塞。下面是相关源码：

```c

void clientsCron(void) {
    /* Make sure to process at least 1/(server.hz*10) of clients per call.
     *
     * 这个函数每次执行都会处理至少 1/server.hz*10 个客户端。
     *
     * Since this function is called server.hz times per second we are sure that
     * in the worst case we process all the clients in 10 seconds.
     *
     * 因为这个函数每秒钟会调用 server.hz 次，
     * 所以在最坏情况下，服务器需要使用 10 秒钟来遍历所有客户端。
     *
     * In normal conditions (a reasonable number of clients) we process
     * all the clients in a shorter time. 
     *
     * 在一般情况下，遍历所有客户端所需的时间会比实际中短很多。
     */

    // 客户端数量
    int numclients = listLength(server.clients);

    // 要处理的客户端数量
    int iterations = numclients / (server.hz * 10);

    // 至少要处理 50 个客户端
    if (iterations < 50)
        iterations = (numclients < 50) ? numclients : 50;

    while (listLength(server.clients) && iterations--) {
        redisClient *c;
        listNode *head;

        /* Rotate the list, take the current head, process.
         * This way if the client must be removed from the list it's the
         * first element and we don't incur into O(N) computation. */
        // 翻转列表，然后取出表头元素，这样一来上一个被处理的客户端会被放到表头
        // 另外，如果程序要删除当前客户端，那么只要删除表头元素就可以了
        listRotate(server.clients);
        head = listFirst(server.clients);
        c = listNodeValue(head);
        /* The following functions do different service checks on the client.
         * The protocol is that they return non-zero if the client was
         * terminated. */
        // 检查客户端，并在客户端超时时关闭它
        if (clientsCronHandleTimeout(c)) continue;
        // 根据情况，缩小客户端查询缓冲区的大小
        if (clientsCronResizeQueryBuffer(c)) continue;
    }
}


```

## 阻塞的取消策略

    当程序添加一个新的被阻塞客户端到 server.blocking_keys 字典的链表中时，他将客户端放在链表的最后，而当 handleClientsBlockedOnLists  取消客户端的阻塞时候，它从链表的最前面开始取消阻塞；这个链表形成了一个 FIFO 队列，最先被阻塞的客户端总是最先脱离阻塞状态，Redis 文档称这种模式为先阻塞先服务(FBFS, first-block-first-server)。