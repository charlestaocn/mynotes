[[0拷贝资料]]
### 传统io

```java
while((n = read(diskfd, buf, BUF_SIZE)) > 0)
    write(sockfd, buf , n);
```


> 两次 DMA 拷贝 两次 CPU 拷贝


![](https://pic1.zhimg.com/80/v2-3475e179de7fd24ea59a1e0b0a0dbf40_720w.webp)

### mmap 地址映射 + write()

 **使用 mmap 处理小数据的频繁读写**

==如果 IO 非常频繁，数据却非常小，推荐使用 mmap，以避免 FileChannel 导致的切态问题。例如索引文件的追加写。==

> 并不仅限用于拷贝发送文件，可以有其他用途。

- dma将数据从磁盘拷贝到内核缓冲区 
- 应用程序不需要拷贝数据到用户缓冲，直接使用内核地址映射的虚拟地址进行数据操作，
- 处理完后调用系统write方法由cpu将数据从内核缓冲拷贝到往外写的socket缓冲，
- 再由dma从socket缓冲拷贝到网卡发出去。

>dma 进行 2 次拷贝。cpu 进行 1次 拷贝。

![[c00593f20f8e65c1672b3d18a72c89a9_MD5.webp]]


### sendfile 系统方法

>仅限于传输文件所以叫sendfile ,数据不经过应用程序的用户空间

  - 由dma将数据从磁盘考到内核缓冲，
  - 再由 cpu 拷贝到  socket 缓冲区 ，
  - 最后由dma 考到 网卡发送出去。
  
  >和mmap比少了mmap方法调用的返回和write方法调用。一样的   dma 进行 2 次拷贝。cpu 进行 1次 拷贝。
  
	![[0ec17ef3471327dd9621443cc75b31c2_MD5.webp]]
	
	
### sendfile+ DMA scatter/gather 

- 由dma将数据从磁盘考到内核缓冲，
- 再由 cpu 拷贝 内核缓冲区中所对应的文件描述符给   socket 缓冲区 ，
- 最后由 dma 根据文件描述符直接从  内核缓冲将数据 考到 网卡发送出去。

>	将cpu拷贝数据的过程转换成了传文件描述符。两次dma拷贝 0次cpu拷贝
>
![[a06466acb7d7d091677e629d8daac3f2_MD5.webp]]