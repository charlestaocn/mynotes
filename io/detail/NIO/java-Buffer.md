缓冲区（Buffer）：一个用于特定基本数据类型的容器。由 java.nio 包定义的，所有缓冲区都是 Buffer 抽象类的子类。

![[Buffer.png]]


### 缓冲区存取数据的两个核心方法



```java
/**
获取缓冲区中的数据
**/
 public abstract T get();
```



```java
/*
存入数据到缓冲区中
*/
public abstract Buffer put(T d);
```


### 缓冲区中的四个核心属性

- `capacity`: 容量，表示缓冲区中最大存储数据的容量。一旦声明不能更改。
- `limit`: 界限，表示缓冲区中可以操作数据的大小。（limit 后的数据不能进行读写）
- `position`: 位置，表示缓冲区中正在操作数据的位置。
- `mark`: 标记，表示记录当前 position 的位置。可以通过 `reset()` 恢复到 `mark` 的位置。




![[bufferOp.webp]]

####  直接缓冲区和非直接缓冲区

[[_resources/java-Buffer/96a01139f70ce28ce65e33c998e4c601_MD5.jpeg|Open: buffer.png]]
![[_resources/java-Buffer/96a01139f70ce28ce65e33c998e4c601_MD5.jpeg]]



1) 非直接缓冲区
	通过 `allocate()` 方法分配缓冲区，将缓冲区建立在 ==JVM 的内存==之中。
	
![[2c8f25d8b67948ffd458b53b5e45513c_MD5.webp]]

	应用程序和磁盘之间想要传输数据，是没有办法直接进行传输的。操作系统出于安全的考虑，会经过上图几个步骤。例如，我应用程序想从磁盘中读取一个数据，这时候我应用程序向操作系统发起一个读请求，那么首先磁盘中的数据会被读取到内核地址空间中，然后会把内核地址空间中的数据拷贝到用户地址空间中（其实就是 JVM 内存中），最后再把这个数据读取到应用程序中来。  
	  
	同样，如果我应用程序有数据想要写到磁盘中去，那么它会首先把这个数据写入到用户地址空间中去，然后把数据拷贝到内核地址空间，最后再把这个数据写入到磁盘中去。


```java
ByteBuffer buffer = ByteBuffer.allocate(8192);
```


2) 直接缓冲区

通过 `allocateDirect()` 方法分配缓冲区，将缓冲区建立在物理内存之中
![](https://pic3.zhimg.com/80/v2-7806ea1690116ec297492d668bbd644e_720w.webp)

	直接用物理内存作为缓冲区，读写数据直接通过物理内存进行，省去多次copy。
	
```java
ByteBuffer buffer = ByteBuffer.allocateDirect(4096);
```
	