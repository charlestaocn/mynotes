各自在第一次握手创建`socket`将四元组(双方各自ip和端口号)存入`socket`对象

![[socket.png]]
![[socket 1.png]]


###  1.创建套接字──socket()

```cpp
@desc *用来创建套接字，确定套接字的各种属性，以进行网络通信*

@param
- amily: 套接字家族，可以使`AF_UNIX`或者`AF_INET`。
- type: 套接字类型，根据是面向连接的还是非连接分为`SOCK_STREAM`或`SOCK_DGRAM`，也就是TCP和UDP的区别。
- protocol: 一般不填默认为0。

@return int (fd) 套接字文件描述符


int socket (int af, int type, int protocol);
```


### 2.指定本地地址──bind()


```cpp
@desc *用于服务端将通信的地址和端口绑定到 socket 上。只有这样，流经该 ip地址和端口的数据才能交给该socket处理*

@param
- int sockfd： 用来标识服务端套接字，由socket()返回的文件描述符。
- struct sockaddr *addr： 用来保存服务端套接字信息(包括IP和端口等)。
- socklen_t *addrlen： 表示addr地址空间大小，与int*类型一样，可由 sizeof() 计算得出。

@return resCode 返回0。否则返回SOCKET_ERROR。

int  bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
``` 


### 3. ==客户端使用==与服务端建立连接connect()

	

```cpp
@desc *用于客户端使用与服务端建立连接，参数与bind()函数参数完全一致。*

@param
- int sockfd： 用来标识服务端套接字，由socket函数返回的文件描述符。
- struct sockaddr *addr： 用来保存服务端套接字信息(包括IP和端口等)。
- socklen_t *addrlen： 表示addr地址空间大小，与int*类型一样，可由 sizeof() 计算得出。
- 
@return resCode 返回0。否则返回ERROR。

int connect(int sock, struct sockaddr *serv_addr, socklen_t addrlen);
```


###  4.监听连接 listen()

```cpp
@desc *把一个未连接的套接字转换成被动套接字,使其可以接受来自其他主动套接字的连接请求，并限制Server程序调用accept()之前的最大连接数*。


@param 
- int sockfd： 由socket函数返回，要被listen函数作用的套接字文件描述符。
- int backlog： 内核进程在自己的空间里维护的一个跟踪已完成连接但服务器进程还没有接手处理或正在进行的连接的队列的大小。backlog告诉内核使用这个数值作为队列大小的上限。

@return
	- 成功:0
	- 失败:-1
int listen(int sockfd, int backlog);
```

### 5. 数据传输──send()与recv()
```cpp
@desc *套接字发送函数*
@param
- int sockfd：套接字文件描述符
- void buf：发送缓冲区
- size_t len：发送数据长度
- int flags

@return
- 成功:返回发送的字节数
- 失败:返回-1，并设置errno
int  send(int sockfd, const void *buf, size_t len, int flags);

```


### 6.接受数据 -- recv()

```c

@desc 接受数据

@param

- int sockfd 第一个参数指定接收端套接字描述符； 
- void  指明一个缓冲区，该缓冲区用来存放recv函数接收到的数据； 
- size_t 指明buf的长度；
- int 一般置0。

@return
- 成功:返回接收到的字节数。
- 其他为失败

int recv(int sockfd, void *buf, size_t len, int flags);
```