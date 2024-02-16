
- Channel是一种新的IO的访问方式，用于在字节缓冲区与通道另一侧的实体（==可以是文件，也可以是Socket）==之间进行传输数据
- Channel可以==双向读写数据，也可以实现异步读写==。全双工，可以并行读写
- 程序不能直接访问Channel，==Channel只能与Buffer缓冲区进行交互==，即把通道中的数据读到Buffer缓冲区中，程序从缓冲区中读取数据；在写操作时，程序把数据写入Buffer缓冲区中，再把缓冲区的数据写入到Channel中



|区别|Stream(传统io)|Channel|
|---|---|---|
|支持异步|不支持|支持|
|是否可双向传输数据|不能，只能单向|可以，既可以从通道读取数据，也可以向通道写入数据|
|是否结合Buffer使用|不|必须结合Buffer使用|
|性能|较低|较高|


常见的Channel
- FileChannel：读写文件的通道
- SocketChannel/ServerSocketChannel：读写Socket套接字中的数据
- DatagramChannel：通过UDP读写网络数据

总之
1) 不管是哪个Channel，都不能通过构造器创建Channel
2) 通道不能重复使用，打开一个通道，表示一个特定的IO服务来建立一个连接，通道关闭时，连接就没了
3) 通道可以阻塞或者非阻塞方式运行



```java
public interface Channel extends Closeable {
	// 通道是否处于开启状态
    public boolean isOpen();
	//用来关闭通道
    public void close() throws IOException;

}


public interface ReadableByteChannel extends Channel {
		public int read(ByteBuffer dst) throws IOException;
}


public interface WritableByteChannel  extends Channel{
		public int write(ByteBuffer src) throws IOException;
}

```
![[_resources/Channel/8562cf5621a30d80cd1f8c2790384cda_MD5.png]]


### FileChannel

- FileChannel通过RandomAccessFile、FileInputStream、FileOutputStream对象**调用getChannel()方法获取**
- FileChannel虽然是双向的，既可以读也可以写，但是从FileInputStream流中获得的通道只能读不能写，如果进行写操作或抛出异常。从FileOutputStream流中获得的通道只能写不能读
- 如果访问的文件是只读的，也不能执行写操作
- FileChannel是线程安全的，但是并不是所有的操作都是多线程的，如影响通道位置或者影响文件大小的操作都是单线程的


```java

//读取数据
RandomAccessFile aFile = new RandomAccessFile("d:\\soft\\nio-data.txt", "rw");  
FileChannel channel = aFile.getChannel();  
ByteBuffer buf = ByteBuffer.allocate(48);  
int bytesRead = channel.read(buf);  
while (bytesRead != -1) {  
    System.out.println("Read " + bytesRead);  
    buf.flip();  
    while (buf.hasRemaining()) {  
        System.out.print((char) buf.get());  
    }  
  
    buf.clear();  
    bytesRead = channel.read(buf);  
}  
aFile.close();  
System.out.println("------");


//写数据

String newData = "New String to write to file..." + System.currentTimeMillis();  
ByteBuffer buf = ByteBuffer.allocate(48);  
buf.clear();  
buf.put(newData.getBytes());  
  
buf.flip();  
  
while(buf.hasRemaining()) {  
    channel.write(buf);  
}



```
****


###  SocketChannel和ServerSocketChannel

- ServerSocketChannel可以监听新进来的TCP通道 (==ServerSocketChannel 默认时阻塞式的:`accept()` `read()`  会阻塞线程。 使用`SocketChannel..configureBlocking(false)` 将 asc 设置为非阻塞模式，这时在调用 `read()` 方法就不会阻塞了，如果没有可读数据，它则会返回 -1)==
- SocketChannel是一个连接到TCP网络套接字的通道

![[d3f2ce2311b99e53e71539c8a82ab275_MD5.jpg]]


[[_resources/Channel/f04b9ed79d8b00a3559b9235b7396dc6_MD5.jpg|Open: WeChat94e7f81abad904708513ecb30b7bbe97.jpg]]
![[_resources/Channel/f04b9ed79d8b00a3559b9235b7396dc6_MD5.jpg]]


```java

InetSocketAddress serverAddr = new InetSocketAddress("192.168.150.11", 9090);
SocketChannel client1 = SocketChannel.open();
//bing 本地地址端口
client1.bind(new InetSocketAddress("192.168.150.1", 8080));
//连接服务器
client1.connect(serverAddr);
//向服务端写数据
client1.write(writeBuffer);
//读取服务端数据
client2.read(readBuffer);
```




## 如何将Channel注册到Selector上？


只有将Channel注册到Selector之后才能被Selector轮询，注册代码如下：
```java
//创建Reactor线程，创建多路复用器并启动线程 
selector = Selector.open(); 
//打开ServerSocketChannel，用于监听客户端的连接，它是所有客户端连接的父管道，是SelectableChannel（负责网络读写）的子类 
serverChannel = ServerSocketChannel.open(); 
//设置Channel为非阻塞模式 
serverChannel.configureBlocking(false); 
//绑定端口 
serverChannel.socket().bind(new InetSocketAddress(port), 1024);
//将Channel注册到selector上 
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

```

`serverChannel.register(selector, SelectionKey.OP_ACCEPT)`第二个参数表示一个“inerest集合”，意思是通过Selector监听Channel时，对哪些事件感兴趣，可以是多个，可以监听的事件类型如下，同时可以通过再次调用register方法来改变感兴趣的事件。

- CONNECT：连接完成事件，仅适用于客户端
- ACCEPT：接受新连接事件，仅适用于服务端
- READ：读事件，适用于两端，表示可读
- WRITE：写事件，适用于两端，表示可写

	-  一个客户端Channel成功连接到另一个服务器，称为连接就绪
	- 一个ServerSocketChannel准备好接收新进入的连接，称为接收就绪
	- 一个有数据可读的Channel，是读就绪
	- 一个等待写数据的Channel，是写就绪`

## SelectionKey

SelectionKey是一个抽象类，表示一个Channel和一个Selector的关系。[](https://link.juejin.cn?target=)

### API

- `#channel`：返回绑定的Channel
- `#selector`：获取绑定的Selector
- `#interestOps`：获取感兴趣的事件集合
- `#readyOps`：获取就绪的事件集合

### 获取方式

- `serverChannel.register`，将Channel注册到Selector后会返回`Channel`和`Selector`的SelectionKey
- 通过Selector可以获取，`Selector#selectedKeys()`，可以获取到当前Selector上注册的所有的SelectionKey

### 使用示例

[[_resources/Channel/d08b0930301c31181f3a9a273fbd118e_MD5.jpg|Open: WeChat461a9a536b7ffd5c3400a76db10028fd.jpg]]
![[_resources/Channel/d08b0930301c31181f3a9a273fbd118e_MD5.jpg]]



```java
server = ServerSocketChannel.open();  
server.configureBlocking(false);  
server.bind(new InetSocketAddress(port));  
selector = Selector.open();  // java 封装 （select  poll  *epoll）  =》 selector
server.register(selector, SelectionKey.OP_ACCEPT);  
Set<SelectionKey> selectionKeys = selector.selectedKeys();  
Iterator<SelectionKey> iter = selectionKeys.iterator();  
while (iter.hasNext()) {  
    SelectionKey key = iter.next();  
    iter.remove();  
    if (key.isAcceptable()) {  
  
        //接受连接   
ServerSocketChannel ssc = (ServerSocketChannel) key.channel();  
        SocketChannel client = ssc.accept();  
        client.register(selector, SelectionKey.OP_READ, buffer);  
  
    } else if (key.isReadable()) {  
        //读取数据  
        SocketChannel client = (SocketChannel) key.channel();  
        ByteBuffer buffer = (ByteBuffer) key.attachment();  
        read = client.read(buffer);  
    } else if (key.isWritable()) {  
        SocketChannel client = (SocketChannel) key.channel();  
        ByteBuffer buffer = (ByteBuffer) key.attachment();  
        client.write(buffer);
    }  
}

```
