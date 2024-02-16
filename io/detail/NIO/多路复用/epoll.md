![[epoll-process.webp]]
### 1. 函数

#### 1.1 `epoll_create`

 
 应用程序调用内核系统调用，开辟内存空间

创建一个epoll的句柄，size用来告诉内核监听的文件描述符数目的大小。这个参数不同于`select()`中的第一个参数（select第一个参数()给出最大监听的fd号+1的值）。

当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用`close()`关闭，否则可能导致fd被耗尽。


```c

@param
int size 内核监听的文件描述符数目的大小

@return  int epfd

int epoll_create(int size);
```



#### 1.2 `epoll_ctl`


应用程序新连接，注册到对应的内核空间

epoll的事件注册函数，它不同与`select()`是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型

```c

@param 

- int epfd epoll_create()的返回值 ，
- int op ，用三个宏来表示 ：
		-EPOLL_CTL_ADD：注册新的fd到epfd中；
		-EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
		-EPOLL_CTL_DEL：从epfd中删除一个fd；
		
- int fd 是需要监听的fd 
- struct epoll_event 是告诉内核需要监听什么事件。

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

```
#### 1.3 `epoll_wait`


应用程序询问需要处理的连接 
		- 哪些需要处理？有读，写，错误的事件
	 

`epoll_wait`等侍注册在`epfd`上的`socket fd`的事件的发生，如果发生则将发生的`sokct fd`和事件类型放入到`events`数组中。**并且将注册在`epfd`上的`socket fd`的事件类型给清空**.

```c

@param
- epfd:由epoll_create 生成的epoll专用的文件描述符；
- events:用于存储待处理事件的数组； 
- maxevents:events数组中成员的个数； 
- timeout:等待I/O事件发生的超时值；-1相当于阻塞，0相当于非阻塞。一般用-1即可

@return
- 返回需要处理的事件数目，
- 如返回0表示已超时。
	
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

```

#### 1.4 epoll各函数调用顺序

1. 先是使用 `epoll_create(int size)`：在内存中创建一个指定size大小的事件空间。
2. 再使用`epoll_ctl`事件注册函数：注册新的fd到epfd中，并指明event(可读写啊等等），注意：在注册新事件fd的过程中，也在内核中断处理程序里注册fd对应的回调函数callback，告诉内核，一旦这个fd中断了，就把它放到ready队列里面去。
3. 再使用`epoll_wait`：在epoll对象对应的活跃事件数组events里取就绪的fd，并使用内存映射mmap拷贝到用户空间。
4. 再在用户空间依次处理相关的fd。

### 2. epoll的详细工作流程


#### 2.1 创建`epoll`对象

如下图所示，当某个进程调用`epoll_create`方法时，内核会创建一个`eventpoll`对象（也就是程序中epfd所代表的对象）。`eventpoll`对象也是文件系统中的一员，和socket一样，它也会有等待队列。创建一个代表该epoll的eventpoll对象是必须的，因为内核要维护 就绪列表（rdlist）等数据 。

![[epoll_create.png]]

#### 2.2维护监视列表

创建`epoll`对象后，可以用`epoll_ctl`添加或删除所要监听的socket。以添加socket为例，如下图，如果通过`epoll_ctl`添加sock1、sock2和sock3的监视，内核会将eventpoll添加到这三个socket的等待队列中。当socket收到数据后，中断程序会操作eventpoll对象，而不是直接操作进程。

![[epoll_create.png]]

#### 2.3接受数据

当socket收到数据后，中断程序会给eventpoll的rdlist添加socket引用。如下图展示的是sock2和sock3收到数据后，中断程序让rdlist引用这两个socket。eventpoll对象相当于是socket和进程之间的中介，socket的数据接收并不直接影响进程，而是通过改变eventpoll的就绪列表来改变进程状态。当程序执行到`epoll_wait`时，如果rdlist已经引用了socket，那么`epoll_wait`直接返回，如果rdlist为空，阻塞进程。

![[epol_wait.png]]
#### 2.3阻塞和唤醒进程

假设计算机中正在运行进程A和进程B，在某时刻进程A运行到了`epoll_wait`语句。如下图所示，内核会将进程A放入`eventpoll`的等待队列中，阻塞进程。

![[epol_add_rdlist.png]]

当socket接收到数据，中断程序一方面修改rdlist，另一方面唤醒eventpoll等待队列中的进程，进程A再次进入运行状态（如下图）。也因为rdlist的存在，进程A可以知道哪些socket发生了变化。

![[epoll_changeStatus.png]]



### 3. epoll的实现细节

如下图所示，eventpoll包含了lock、mtx、wq（等待队列）、rdlist（就绪队列）、rbr（索引结构）成员。rdlist和rbr是比较重要的两个成员。

![[epoll_detail.png]]

![[epoll.png]]


rdlist引用着就绪的socket，所以它应能够快速的插入数据。程序可能随时调用epoll_ctl添加监视socket，也可能随时删除。当删除时，若该socket已经存放在就绪列表中，它也应该被移除。所以rdlist应是一种能够快速插入和删除的数据结构。双向链表就是这样一种数据结构，epoll使用双向链表来实现就绪队列（对应上图的rdllist）。
epoll使用rbr来保存监视的socket，rbr至少要方便的添加和移除，还要便于搜索，以避免重复添加。
[[数据结构/红黑树]]
### 4. 优缺点

1. 没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）；
2. 效率提升，不是轮询的方式，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数；即Epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，Epoll的效率就会远远高于select和poll。
3. epoll通过epoll_ctl 在内核建立了一个数据结构，该接口会将要监视的文件描述符记录下来，之后调用epoll_wait时就不用再传递任何文件描述符的信息给内核了，但是每次添加文件描述符的时候需要执行一次系统调用。即select每次传入和取回时都要涉及到fd数组在用户态和内核态之间的复制，而epoll只复制一次
### 5. 为何`epoll_wait`复杂度为O(1)

当socket就绪时，中断程序会操作eventPoll，在eventPoll中的就绪列表(rdlist)，添加scoket引用。这样的话，进程A只需要不断循环遍历rdlist，从而获取就绪的socket，而不会遍历所有的socket。

	- poll和select的时间复杂度为O(n)：每个监听的文件描述符都遍历一遍。
	- epoll的时间复杂度为O（1）：只遍历就绪的socket，内核中断程序会添加就绪的socket到rdlist。



### 6. java nio(new io)使用


