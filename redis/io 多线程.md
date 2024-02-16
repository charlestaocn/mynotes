### **一、多线程 Redis 服务启动**


默认情况下多线程是默认关闭的。如果想要启动多线程，需要在配置文件中做适当的修改。相关的配置项是 io-threads 和 io-threads-do-reads 两个。

```javascript
#vi /usr/local/soft/redis6/conf/redis.conf 
io-threads 4 #启用的 io 线程数量
io-threads-do-reads yes #读请求也使用io线程
```



其中 io-threads 表示要启动的 io 线程的数量。io-threads-do-reads 表示是否在读阶段也使用 io 线程，默认是只在写阶段使用 io 线程的。

现在假设我们已经打开了如上两项多线程配置。带着这个假设，让我们进入到 Redis 的 main 入口函数。

```javascript
//file: src/server.c
int main(int argc, char **argv) {
    ......

    // 1.1 主线程初始化
    initServer();

    // 1.2 启动 io 线程
    InitServerLast();

    // 进入事件循环
    aeMain(server.el);
}
```


#### **1.1 主线程初始化**

在 initServer 这个函数内，Redis 主线程做了这么几件重要的事情。

![[redis/_resources/io 多线程/c71a9f9f2d12b23b076fee970642ad3f_MD5.png]]

- 初始化读任务队列、写任务队列
- 创建一个 epoll 对象
- 对配置的监听端口进行 listen
- 把 listen socket 让 epoll 给管理起来

```javascript
//file: src/server.c
void initServer() {

    // 1 初始化 server 对象
    server.clients_pending_write = listCreate();
    server.clients_pending_read = listCreate();
    ......

    // 2 初始化回调 events，创建 epoll
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);

    // 3 绑定监听服务端口
    listenToPort(server.port,server.ipfd,&server.ipfd_count);

    // 4 注册 accept 事件处理器
    for (j = 0; j < server.ipfd_count; j++) {
        aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL);
    }
    ...
}
```



接下来我们分别来看。

##### **初始化 server 对象**

在 initServer 的一开头，先是对 server 的各种成员变量进行初始化。值得注意的是 clients_pending_write 和 clients_pending_read 这两个成员，它们分别是写任务队列和读任务队列。将来主线程产生的任务都会放在放在这两个任务队列里。

主线程会根据这两个任务队列来进行任务哈希散列，以将任务分配到多个线程中进行处理。

##### **aeCreateEventLoop 处理**

我们来看 aeCreateEventLoop 详细逻辑。它会初始化事件回调 event，并且创建了一个 epoll 对象出来。

```javascript
//file:src/ae.c
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    eventLoop = zmalloc(sizeof(*eventLoop);

    //将来的各种回调事件就都会存在这里
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    ......

    aeApiCreate(eventLoop);
    return eventLoop;
}
```



我们注意一下 eventLoop->events，将来在各种事件注册的时候都会保存到这个数组里。

```javascript
//file:src/ae.h
typedef struct aeEventLoop {
    ......
    aeFileEvent *events; /* Registered events */
}
```



具体创建 epoll 的过程在 ae_epoll.c 文件下的 aeApiCreate 中。在这里，真正调用了 epoll_create

```javascript
//file:src/ae_epoll.c
static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(aeApiState));
    state->epfd = epoll_create(1024); 
    eventLoop->apidata = state;
    return 0;
}
```



##### **绑定监听服务端口**

我们再来看 Redis 中的 listen 过程，它在 listenToPort 函数中。调用链条很长，依次是 listenToPort => anetTcpServer => _anetTcpServer => anetListen。在 anetListen 中，就是简单的 bind 和 listen 的调用。

```javascript
//file:src/anet.c
static int anetListen(......) {
    bind(s,sa,len);
    listen(s, backlog);
    ......
}
```



##### **注册事件回调函数**

前面我们调用 aeCreateEventLoop 创建了 epoll，调用 listenToPort 进行了服务端口的 bind 和 listen。接着就调用的 aeCreateFileEvent 就是来注册一个 accept 事件处理器。

我们来看 aeCreateFileEvent 具体代码。

```javascript
//file: src/ae.c
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    // 取出一个文件事件结构
    aeFileEvent *fe = &eventLoop->events[fd];

    // 监听指定 fd 的指定事件
    aeApiAddEvent(eventLoop, fd, mask);

    // 设置文件事件类型，以及事件的处理器
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;

    // 私有数据
    fe->clientData = clientData;
}
```



函数 aeCreateFileEvent 一开始，从 eventLoop->events 获取了一个 aeFileEvent 对象。

接下来调用 aeApiAddEvent。这个函数其实就是对 epoll_ctl 的一个封装。主要就是实际执行 epoll_ctl EPOLL_CTL_ADD。

```javascript
//file:src/ae_epoll.c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    // add or mod
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;
    ......

    // epoll_ctl 添加事件
    epoll_ctl(state->epfd,op,fd,&ee);
    return 0;
}
```



每一个 eventLoop->events 元素都指向一个 aeFileEvent 对象。在这个对象上，设置了三个关键东西

- rfileProc：读事件回调
- wfileProc：写事件回调
- clientData：一些额外的扩展数据

将来 当 epoll_wait 发现某个 fd 上有事件发生的时候，这样 redis 首先根据 fd 到 eventLoop->events 中查找 aeFileEvent 对象，然后再看 rfileProc、wfileProc 就可以找到读、写回调处理函数。

回头看 initServer 调用 aeCreateFileEvent 时传参来看。

```javascript
//file: src/server.c
void initServer() {
    ......

    for (j = 0; j < server.ipfd_count; j++) {
        aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL);
    }
}
```



listen fd 对应的读回调函数 rfileProc 事实上就被设置成了 acceptTcpHandler，写回调没有设置，私有数据 client_data 也为 null。

#### **1.2 io 线程启动**

在主线程启动以后，会调用 InitServerLast => initThreadedIO 来创建多个 io 线程。

![[redis/_resources/io 多线程/970f0ae63f938a876f8f6cab41ce9f30_MD5.png]]

将来这些 IO 线程会配合主线程一起共同来处理所有的 read 和 write 任务。

![[redis/_resources/io 多线程/6dda94fcb2d16cdbf0c0490bfca108c9_MD5.png]]

我们来看 InitServerLast 创建 IO 线程的过程。

```javascript
//file:src/server.c
void InitServerLast() {
    initThreadedIO();
    ......
}
```



```javascript
//file:src/networking.c
void initThreadedIO(void) {
    //如果没开启多 io 线程配置就不创建了
    if (server.io_threads_num == 1) return;

    //开始 io 线程的创建
    for (int i = 0; i < server.io_threads_num; i++) {
        pthread_t tid;
        pthread_create(&tid,NULL,IOThreadMain,(void*)(long)i)
        io_threads[i] = tid;
    }
}
```



在 initThreadedIO 中调用 pthread_create 库函数创建线程，并且注册线程回调函数 IOThreadMain。

```javascript
//file:src/networking.c
void *IOThreadMain(void *myid) {
    long id = (unsigned long)myid;

    while(1) {
        //循环等待任务
        for (int j = 0; j < 1000000; j++) {
            if (getIOPendingCount(id) != 0) break;
        }

        //允许主线程来关闭自己
        ......

        //遍历当前线程等待队列里的请求 client
        listIter li;
        listNode *ln;
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn);
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        listEmpty(io_threads_list[id]);
    }
}
```



是将当前线程等待队列 io_threads_list[id] 里所有的请求 client，依次取出处理。其中读操作通过 readQueryFromClient 处理， 写操作通过 writeToClient 处理。其中 io_threads_list[id] 中的任务是主线程分配过来的，后面我们将会看到。

### **二、主线程事件循环**

接着我们进入到 Redis 最重要的 aeMain，这个函数就是一个死循环（Redis 不退出的话），不停地执行 aeProcessEvents 函数。

```javascript
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```



其中 aeProcessEvents 就是所谓的事件分发器。它通过调用 epoll_wait 来发现所发生的各种事件，然后调用事先注册好的处理函数进行处理。

![[redis/_resources/io 多线程/81104739941da9d056054826dda4772b_MD5.png]]

接着看 aeProcessEvents 函数。

```javascript
//file:src/ae.c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    // 2.3 事件循环处理3：epoll_wait 前进行读写任务队列处理
    if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);

    //epoll_wait发现事件并进行处理
    numevents = aeApiPoll(eventLoop, tvp);

    // 从已就绪数组中获取事件
    aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];

    //如果是读事件，并且有读回调函数
    //2.1 如果是 listen socket 读事件，则处理新连接请求
    //2.2 如果是客户连接socket 读事件，处理客户连接上的读请求
    fe->rfileProc()

    //如果是写事件，并且有写回调函数
    fe->wfileProc()
    ......
}
```



其中 aeApiPoll 就是对 epoll_wait 的一个封装而已。

```javascript
//file: src/ae_epoll.c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    // 等待事件
    aeApiState *state = eventLoop->apidata;
    epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    ...
}
```



aeProcessEvents 就是调用 epoll_wait 来发现事件。当发现有某个 fd 上事件发生以后，则调为其事先注册的事件处理器函数 rfileProc 和 wfileProc。

#### **2.1 事件循环处理1：新连接到达**

在 1.1 节中我们看到，主线程初始化的时候，将 listen socket 上的读事件处理函数注册成了 acceptTcpHandler。也就是说如果有新连接到达的时候，acceptTcpHandler 将会被执行到。

![[redis/_resources/io 多线程/d5dc7df7b6fbfcd68d8bc1ac606af1c7_MD5.png]]

在这个函数内，主要完成如下几件事情。

- 调用 accept 接收连接
- 创建一个 redisClient对象
- 添加到 epoll
- 注册读事件处理函数

接下来让我们进入 acceptTcpHandler 源码。

```javascript
//file:src/networking.c
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    ......
    cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
    acceptCommonHandler(connCreateAcceptedSocket(cfd),0,cip);
}
```



其中 netTcpAccept 调用 accept 系统调用获取连接，就不展开了。我们看 acceptCommonHandler。

```javascript
//file: src/networking.c
static void acceptCommonHandler(int fd, int flags) {
    // 创建客户端
    redisClient *c;
    if ((c = createClient(fd)) == NULL) {
    }
}

client *createClient(connection *conn) {
    client *c = zmalloc(sizeof(client));

    // 为用户连接注册读事件处理器
    if (conn) {
        ...
        connSetReadHandler(conn, readQueryFromClient);
        connSetPrivateData(conn, c);
    }

    selectDb(c,0);
    c->id = client_id;
    c->resp = 2;
    c->conn = conn;
    ......
}
```



在上面的代码中，我们重点关注 `connSetReadHandler(conn, readQueryFromClient)`, 这一行是将这个新连接的读事件处理函数设置成了 readQueryFromClient。

#### **2.2 事件循环处理2：用户命令请求到达**

在上面我们看到了， Redis 把用户连接上的读请求处理函数设置成了 readQueryFromClient，这意味着当用户连接上有命令发送过来的时候，会进入 readQueryFromClient 开始执行。

在多线程版本的 readQueryFromClient 中，处理逻辑非常简单，仅仅只是将发生读时间的 client 放到了任务队列里而已。

![[redis/_resources/io 多线程/832881aa7bd0f3f1f5ec2ef77f3e3188_MD5.png]]

来详细看 readQueryFromClient 代码。

```javascript
//file:src/networking.c
void readQueryFromClient(connection *conn) {
    client *c = connGetPrivateData(conn);

    //如果启动 threaded I/O 的话，直接入队
    if (postponeClientRead(c)) return;

    //处理用户连接读请求
    ......
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    nread = connRead(c->conn, c->querybuf+qblen, readlen);
    processInputBuffer(c);
}
```



在 postponeClientRead 中判断，是不是开启了多 io 线程，如果开启了的话，那就将有请求数据到达的 client 直接放到读任务队列（server.clients_pending_read）中就算是完事。我们看下 postponeClientRead。

```javascript
//file:src/networking.c
int postponeClientRead(client *c) {
    if (server.io_threads_active &&
        server.io_threads_do_reads &&
        !ProcessingEventsWhileBlocked &&
        !(c->flags & (CLIENT_MASTER|CLIENT_SLAVE|CLIENT_PENDING_READ)))
    {
        c->flags |= CLIENT_PENDING_READ;
        listAddNodeHead(server.clients_pending_read,c);
        return 1;
    } else {
        return 0;
    }
}
```



listAddNodeHead 就是把这个 client 对象添加到 server.clients_pending_read 而已。

#### **2.3 事件循环处理3：epoll_wait 前进行任务处理**

在 aeProcessEvents 中假如 aeApiPoll(epoll_wait)中的事件都处理完了以后，则会进入下一次的循环再次进入 aeProcessEvents。

而这一次中 beforesleep 将会处理前面读事件处理函数添加的读任务队列了。

```javascript
//file:src/ae.c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    // 参见 2.4 事件循环处理3：epoll_wait 前进行任务处理
    if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);

    //epoll_wait发现事件并进行处理
    numevents = aeApiPoll(eventLoop, tvp);
    ......
}
```



在 beforeSleep 里会依次处理两个任务队列。先处理读任务队列，解析其中的请求，并处理之。然后将处理结果写到缓存中，同时写到写任务队列中。紧接着 beforeSleep 会进入写任务队列处理，会将处理结果写到 socket 里，进行真正的数据发送。

![[redis/_resources/io 多线程/2c0ee843b456501a8ad7974fabc7bd5e_MD5.png]]

我们来看 beforeSleep 的代码，这个函数中最重要的两个调用是 handleClientsWithPendingReadsUsingThreads（处理读任务队列），handleClientsWithPendingWritesUsingThreads（处理写任务队列）

```javascript
//file:src/server.c
void beforeSleep(struct aeEventLoop *eventLoop) {
    //处理读任务队列
    handleClientsWithPendingReadsUsingThreads();
    //处理写任务队列
    handleClientsWithPendingWritesUsingThreads();
    ......
}
```



值得注意的是，如果开启了多 io 线程的话，handleClientsWithPendingReadsUsingThreads 和 handleClientsWithPendingWritesUsingThreads 中将会是主线程、io 线程一起配合来处理的。所以我们单独分两个小节来阐述。

#### **三、主线程 && io 线程处理读请求**

在 handleClientsWithPendingReadsUsingThreads 中，主线程会遍历读任务队列 server.clients_pending_read，把其中的请求分配到每个 io 线程的处理队列 io_threads_list[target_id] 中。然后通知各个 io 线程开始处理。

![[redis/_resources/io 多线程/b291435ce7168ca40439052e82f195b3_MD5.png]]

#### **3.1 主线程分配任务**

我们来看 handleClientsWithPendingReadsUsingThreads 详细代码。

```javascript
//file:src/networking.c
//当开启了 reading + parsing 多线程 I/O 
//read handler 仅仅只是把 clients 推到读队列里
//而这个函数开始处理该任务队列
int handleClientsWithPendingReadsUsingThreads(void) {

    //访问读任务队列 server.clients_pending_read
    listRewind(server.clients_pending_read,&li);

    //把每一个任务取出来
    //添加到指定线程的任务队列里 io_threads_list[target_id]
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    //启动Worker线程，处理读请求
    io_threads_op = IO_THREADS_OP_READ;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }

    //主线程处理 0 号任务队列
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        //需要先干掉 CLIENT_PENDING_READ 标志
        //否则 readQueryFromClient 并不处理，而是入队
        client *c = listNodeValue(ln);
        readQueryFromClient(c->conn);
    }

    //主线程等待其它线程处理完毕
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;
    }

    //再跑一遍任务队列，目的是处理输入
    while(listLength(server.clients_pending_read)) {
        ......
        processInputBuffer(c);
        if (!(c->flags & CLIENT_PENDING_WRITE) && clientHasPendingReplies(c))
            clientInstallWriteHandler(c);
    }
}
```



在主线程中将任务分别放到了 io_threads_list 的第 0 到第 N 个元素里。并对 1 : N 号线程通过 setIOPendingCount 发消息，告诉他们起来处理。这时候 io 线程将会在 IOThreadMain 中收到消息并开始处理读任务。

```javascript
//file:src/networking.c
void *IOThreadMain(void *myid) {
    while(1) {
        //遍历当前线程等待队列里的请求 client
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn);
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
    }
}
```



在 io 线程中，从自己的 io_threads_list[id] 中遍历获取待处理的 client。如果发现是读请求处理，则进入 readQueryFromClient 开始处理特定的 client。

而主线程在分配完 1 ：N 任务队列让其它 io 线程处理后，自己则开始处理第 0 号任务池。同样是会进入到 readQueryFromClient 中来执行。

```javascript
//file:src/networking.c
int handleClientsWithPendingReadsUsingThreads(void) {
    ......
    //主线程处理 0 号任务队列
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        //需要先干掉 CLIENT_PENDING_READ 标志
        //否则 readQueryFromClient 并不处理，而是入队
        client *c = listNodeValue(ln);
        readQueryFromClient(c->conn);
    }
    ......
}
```



所以无论是主线程还是 io 线程，处理客户端的读事件都是会进入 readQueryFromClient。我们来看其源码。

#### **3.2 读请求处理**

```javascript
//file:src/networking.c
void readQueryFromClient(connection *conn) {

    //读取请求
    nread = connRead(c->conn, c->querybuf+qblen, readlen);

    //处理请求
    processInputBuffer(c);
}
```



在 connRead 中就是调用 read 将 socket 中的命令读取出来，就不展开看了。接着在 processInputBuffer 中将输入缓冲区中的数据解析成对应的命令。解析完命令后真正开始处理它。

```javascript
//file:src/networking.c
void processInputBuffer(client *c) {
    while(c->qb_pos < sdslen(c->querybuf)) {
        //解析命令
        ......
        //真正开始处理 command
        processCommandAndResetClient(c);
    }
}
```



函数 processCommandAndResetClient 会调用 processCommand，查询命令并开始执行。执行的核心方法是 call 函数，我们直接看它。

```javascript
//file:src/server.c
void call(client *c, int flags) {

    // 查找处理命令，
    struct redisCommand *real_cmd = c->cmd;

    // 调用命令处理函数
    c->cmd->proc(c);

    ......
}
```



在 server.c 中定义了每一个命令对应的处理函数

```javascript
//file:src/server.c
struct redisCommand redisCommandTable[] = {
    {"module",moduleCommand,-2,"as",0,NULL,0,0,0,0,0},
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    {"setnx",setnxCommand,3,"wmF",0,NULL,1,1,1,0,0},
    {"setex",setexCommand,4,"wm",0,NULL,1,1,1,0,0},
    ......

    {"mget",mgetCommand,-2,"rF",0,NULL,1,-1,1,0,0},
    {"rpush",rpushCommand,-3,"wmF",0,NULL,1,1,1,0,0},
    {"lpush",lpushCommand,-3,"wmF",0,NULL,1,1,1,0,0},
    {"rpushx",rpushxCommand,-3,"wmF",0,NULL,1,1,1,0,0},
    ......
}
```



对于 get 命令来说，其对应的命令处理函数就是 getCommand。也就是说当处理 GET 命令执行到 `c->cmd->proc` 的时候会进入到 getCommand 函数中来。

```javascript
//file: src/t_string.c
void getCommand(client *c) {
    getGenericCommand(c);
}
int getGenericCommand(client *c) {
    robj *o;

    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.null[c->resp])) == NULL)
        return C_OK;
    ...
    addReplyBulk(c,o);
    return C_OK;
}
```



getGenericCommand 方法会调用 lookupKeyReadOrReply 来从内存中查找对应的 key值。如果找不到，则直接返回 C_OK；如果找到了，调用 addReplyBulk 方法将值添加到输出缓冲区中。

```javascript
//file: src/networking.c
void addReplyBulk(client *c, robj *obj) {
    addReplyBulkLen(c,obj);
    addReply(c,obj);
    addReply(c,shared.crlf);
}
```



#### **3.3 写处理结果到发送缓存区**

其主体是调用 addReply 来设置回复数据。在 addReply 方法中做了两件事情：

- prepareClientToWrite 判断是否需要返回数据，并且将当前 client 添加到等待写返回数据队列中。
- 调用 _addReplyToBuffer 和 _addReplyObjectToList 方法将返回值写入到输出缓冲区中，等待写入 socekt

```javascript
//file:src/networking.c
void addReply(client *c, robj *obj) {
    if (prepareClientToWrite(c) != C_OK) return;

    if (sdsEncodedObject(obj)) {
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            _addReplyStringToList(c,obj->ptr,sdslen(obj->ptr));
    } else {
        ......        
    }
}
```



先来看 prepareClientToWrite 的详细实现，

```javascript
//file: src/networking.c
int prepareClientToWrite(client *c) {
    ......

    if (!clientHasPendingReplies(c) && !(c->flags & CLIENT_PENDING_READ))
        clientInstallWriteHandler(c);
}

//file:src/networking.c
void clientInstallWriteHandler(client *c) {
    c->flags |= CLIENT_PENDING_WRITE;
    listAddNodeHead(server.clients_pending_write,c);
}
```



其中 server.clients_pending_write 就是我们说的任务队列，队列中的每一个元素都是有待写返回数据的 client 对象。在 prepareClientToWrite 函数中，把 client 添加到任务队列 server.clients_pending_write 里就算完事。

接下再来 _addReplyToBuffer，该方法是向固定缓存中写，如果写不下的话就继续调用 _addReplyStringToList 往链表里写。简单起见，我们只看 _addReplyToBuffer 的代码。

```javascript
//file:src/networking.c
int _addReplyToBuffer(client *c, const char *s, size_t len) {
    ......
    // 拷贝到 client 对象的 Response buffer 中
    memcpy(c->buf+c->bufpos,s,len);
    c->bufpos+=len;
    return C_OK;
}
```



要注意的是，本节的读请求处理过程是主线程和 io 线程在并行执行的。主线程在处理完后会等待其它的 io 线程处理。在所有的读请求都处理完后，主线程 beforeSleep 中对 handleClientsWithPendingReadsUsingThreads 的调用就结束了。

#### **四、主线程 && io 线程配合处理写请求**

当所有的读请求处理完后，handleClientsWithPendingReadsUsingThreads 会退出。主线程会紧接着进入 handleClientsWithPendingWritesUsingThreads 中来处理。

![[redis/_resources/io 多线程/1523e659e6ec42b348f629d190dcca40_MD5.png]]

```javascript
//file:src/server.c
void beforeSleep(struct aeEventLoop *eventLoop) {
    //处理读任务队列
    handleClientsWithPendingReadsUsingThreads();
    //处理写任务队列
    handleClientsWithPendingWritesUsingThreads();
    ......
}
```



#### **4.1 主线程分配任务**

```javascript
//file:src/networking.c
int handleClientsWithPendingWritesUsingThreads(void) {
    //没有开启多线程的话，仍然是主线程自己写
    if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
        return handleClientsWithPendingWrites();
    }

    ......

    //获取待写任务
    int processed = listLength(server.clients_pending_write);

    //在N个任务列表中分配该任务
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_write,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;

        /* Remove clients from the list of pending writes since
         * they are going to be closed ASAP. */
        if (c->flags & CLIENT_CLOSE_ASAP) {
            listDelNode(server.clients_pending_write, ln);
            continue;
        }

        //hash的方式进行分配
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    //告诉对应的线程该开始干活了
    io_threads_op = IO_THREADS_OP_WRITE;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }

    //主线程自己也会处理一些
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        writeToClient(c,0);
    }
    listEmpty(io_threads_list[0]);

    //循环等待其它线程结束处理
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;
    }
    ......
}
```



在 io 线程中收到消息后，开始遍历自己的任务队列 io_threads_list[id]，并将其中的 client 挨个取出来开始处理。

```javascript
//file:src/networking.c
void *IOThreadMain(void *myid) {
    while(1) {
        //遍历当前线程等待队列里的请求 client
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn);
            } 
        }
        listEmpty(io_threads_list[id]);
    }
}
```



#### **4.2 写请求处理**

由于这次任务队列里都是写请求，所以 io 线程会进入 writeToClient。而主线程在分配完任务以后，自己开始处理起了 io_threads_list[0]，并也进入到 writeToClient。

```javascript
//file:src/networking.c
int writeToClient(int fd, client *c, int handler_installed) {
    while(clientHasPendingReplies(c)) {
        // 先发送固定缓冲区
        if (c->bufpos > 0) {
            nwritten = write(fd,c->buf+c->sentlen,c->bufpos-c->sentlen);
            if (nwritten <= 0) break;
            ......

        // 再发送回复链表中数据
        } else {
            o = listNodeValue(listFirst(c->reply));
            nwritten = write(fd, o->buf + c->sentlen, objlen - c->sentlen);
            ......
        }
    }
}
```



writeToClient 中的主要逻辑就是调用 write 系统调用让内核帮其把数据发送出去即可。由于每个命令的处理结果大小是不固定的。所以 Redis 采用的做法用固定的 buf + 可变链表来储存结果字符串。这里自然发送的时候就需要分别对固定缓存区和链表来进行发送了。

当所有的写请求也处理完后，beforeSleep 就退出了。主线程将会再次调用 epoll_wait 来发现请求，进入下一轮的用户请求处理。

#### **五、总结**

```javascript
//file: src/server.c
int main(int argc, char **argv) {
    ......

    // 1.1 主线程初始化
    initServer();

    // 1.2 启动 io 线程
    InitServerLast();

    // 进入事件循环
    aeMain(server.el);
}
```



在 initServer 这个函数内，Redis 做了这么三件重要的事情。

- 创建一个 epoll 对象
- 对配置的监听端口进行 listen
- 把 listen socket 让 epoll 给管理起来

在 initThreadedIO 中调用 pthread_create 库函数创建线程，并且注册线程回调函数 IOThreadMain。在 IOThreadMain 中等待其队列 io_threads_list[id] 产生请求，当有请求到达的时候取出 client，依次处理。其中读操作通过 readQueryFromClient 处理， 写操作通过 writeToClient 处理。

主线程在 aeMain 函数中，是一个无休止的循环，它是 Redis 中最重要的部分。它先是调用事件分发器发现事件。如果有新连接请求到达的时候，执行 accept 接收新连接，并为其注册事件处理函数。

当用户连接上有命令请求到达的时候，主线程在 read 处理函数中将其添加到读发送队列中。然后接着在 beforeSleep 中开启对读任务队列和写任务队列的处理。总体工作过程如下图所示。

![[redis/_resources/io 多线程/abecbdcac1f0a8948b59437ce965d376_MD5.png]]

在这个处理过程中，对读任务队列和写任务队列的处理都是多线程并行进行的（前提是开篇我们开启了多 IO 线程并且也并发处理读）。

当读任务队列和写任务队列的都处理完的时候，主线程再一次调用 epoll_wait 去发现新的待处理事件，如此往复循环进行处理。

至此，多线程版本的 Redis 的工作原理就介绍完了。坦白讲，我觉得这种多线程模型实现的并不足够的好。

原因是主线程是在处理读、写任务队列的时候还要等待其它的 io 线程处理完才能进入下一步。假设这时有 10 个用户请求到达，其中 9 个处理耗时需要 1 ms，而另外一个命令需要 1 s。则这时主线程仍然会等待这个 io 线程处理 1s 结束后才能进入后面的处理。整个 Redis 服务还是被一个耗时的命令给 block 住了。

我倒是希望我的理解哪里有问题。因为这种方式真的是没能很好地并发起来


