![[select.png]]


![[select-poll.webp]]


#### 1. 函数
```c
@param
int 	n:最大描述符值 + 1
fd_set *readfds:对可读感兴趣的描述符集
fd_set *writefds:对可写感兴趣的描述符集
fd_set *errorfds:对出错感兴趣的描述符集
struct timeval *timeout:超时时间（注意：对于linux系统，此参数没有const限制，每次select调用完毕timeout的值都被修改为剩余时间，而unix系统则不会改变timeout值）

@return
-1：有错误产生 
0：超时时间到，而且没有描述符有状态变化 
>0：有状态变化的描述符个数


int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

#### 2. 调用:
select函数后程序==会阻塞==，直到有描述副就绪，或者超时当select函数返回后，可以通过==遍历fdset==，来找到就绪的描述符。

### 3. *缺点*

	1. 对socket进行扫描时是线性扫描，即采用轮询的方法，而且遍历所有的socket。效率较低。
	2. 虽然只有一次用户态内核态切换但是传递的文件描述符列表(`fds`)的传递复制开销大。
	3. select 的 fds 参数大小是固定的一个 fd_set(int 1024) 类型的除非改系统设置，不然无法突破

#### 4. 使用实例
```c
uint32 SocketWait(TSocket *s,bool rd,bool wr,uint32 timems)    
{
     fd_set rfds,wfds;
#ifdef _WIN32
     TIMEVAL tv;
#else
     struct timeval tv;
#endif   /* _WIN32 */ 
 
     FD_ZERO(&rfds);
     FD_ZERO(&wfds); 
 
     if (rd)     //TRUE
          FD_SET(*s,&rfds);   //添加要测试的描述字 
     if (wr)     //FALSE
          FD_SET(*s,&wfds); 
 
     tv.tv_sec=timems/1000;     //second
     tv.tv_usec=timems%1000;     //ms 
 
     for (;;) //如果errno==EINTR，反复测试缓冲区的可读性
          switch(select((*s)+1,&rfds,&wfds,NULL,(timems==TIME_INFINITE?NULL:&tv)))  //测试在规定的时间内套接口接收缓冲区中是否有数据可读
         {                                              //0－－超时，-1－－出错
         case 0:     /* time out */
              return 0; 
         case (-1):    /* socket error */
              if (SocketError()==EINTR)
                   break;              
              return 0; //有错但不是EINTR 
          default:
              if (FD_ISSET(*s,&rfds)) //如果s是rfds或wfds中的一员返回非0，否则返回0
                   return 1;
 
              if (FD_ISSET(*s,&wfds))
                   return 2;
              return 0;
         };
}

```