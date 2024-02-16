
![[select-poll.webp]]


### 1、函数
```c

@param

struct pollfd *fds:一个结构体数组，用来保存各个描述符的相关状态。
unsigned long nfds:fdarray数组的大小，即里面包含有效成员的数量。
int timeout:设定的超时时间。（以毫秒为单位）

@return

-1：有错误产生 
0：超时时间到，而且没有描述符有状态变化 
>0：有状态变化的描述符个数

int poll (struct pollfd *fds, unsigned int nfds, int timeout);

```

### 2. 调用
poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态，如果设备就绪则在设备等待队列中加入一项并继续遍历，如果遍历完所有fd后没有发现就绪设备，则挂起当前进程，直到设备就绪或者主动超时。

poll还有一个特点是水平触发，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd。


### 3. 优缺点

   优点：
		它没有select 的数量限制
		
   缺点：
   
		1. 对socket进行扫描时是线性扫描，即采用轮询的方法，而且遍历所有的socket。效率较低。
		2. 虽然只有一次用户态内核态切换但是传递的文件描述符列表(`fds`)的传递复制开销大。

### 4.使用实例
```c
int main(int argc ,char **argv)
{
  struct pollfd fds[IN_FILES];
  char buf[MAX_BUFFER_SIZE];
  int i,res,real_read, maxfd;
  fds[0].fd = 0;
  if((fds[1].fd=open("data1",O_RDONLY|O_NONBLOCK)) < 0)
    {
      fprintf(stderr,"open data1 error:%s",strerror(errno));
      return 1;
    }
  if((fds[2].fd=open("data2",O_RDONLY|O_NONBLOCK)) < 0)
    {
      fprintf(stderr,"open data2 error:%s",strerror(errno));
      return 1;
    }
  for (i = 0; i < IN_FILES; i++)
    {
      fds[i].events = POLLIN;
    }
  while(fds[0].events || fds[1].events || fds[2].events)
    {
      if (poll(fds, IN_FILES, TIME_DELAY) <= 0)
    {
     printf("Poll error\n");
     return 1;
    }
      for (i = 0; i< IN_FILES; i++)
    {
     if (fds[i].revents)
       {
         memset(buf, 0, MAX_BUFFER_SIZE);
         real_read = read(fds[i].fd, buf, MAX_BUFFER_SIZE);
         if (real_read < 0)
        {
         if (errno != EAGAIN)
           {
             return 1;
           }
        }
         else if (!real_read)
        {
         close(fds[i].fd);
         fds[i].events = 0;
        }
         else
        {
         if (i == 0)
           {
             if ((buf[0] == 'q') || (buf[0] == 'Q'))
            {
             return 1;
            }
           }
         else
           {
             buf[real_read] = '\0';
             printf("%s", buf);
           }
        }
       }
    }
    }
  exit(0);
}

```