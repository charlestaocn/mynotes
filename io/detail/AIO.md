（Asynchronous I/O）：异步非阻塞I/O，是一种更高级别的I/O模型。在进行I/O操作时，不需要等待操作完成，就可继续进行其他操作，当操作完成后会自动回调通知

==数据就绪后操作系统直接回调应用程序；==

基本使用和 nio 差不多，但是不需要像nio 那样轮训 `select()`

## 同步写法

```java
   //客户端
        AsynchronousServerSocketChannel assc = AsynchronousServerSocketChannel.open();
        //绑定端口
        assc.bind(new InetSocketAddress(9999));
        //获取连接
        Future<AsynchronousSocketChannel> accept = assc.accept();
        AsynchronousSocketChannel asc = accept.get();
        //创建数组
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        //读取内容长度
        Future<Integer> read = asc.read(buffer);
        Integer len = read.get();
        //获取内容
        System.out.println(new String(buffer.array(),0,len));
        //关闭资源
        asc.close();

```


## 异步非阻塞

```java
//客户端
        AsynchronousServerSocketChannel assc = AsynchronousServerSocketChannel.open();
        //绑定端口
        assc.bind(new InetSocketAddress(9999));
        //获取连接
        assc.accept(123, new CompletionHandler<AsynchronousSocketChannel, Integer>() {
            //成功回调
            @Override
            public void completed(AsynchronousSocketChannel result, Integer attachment) {
                System.out.println("连接成功了 我笑了"+attachment+"秒。");
                //获取连接成功了 开始读取数据
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                result.read(buffer, 99, new CompletionHandler<Integer, Integer>() {
                    @Override
                    public void completed(Integer result, Integer attachment) {
                        System.out.println("读取成功了 我赢了"+attachment+"块钱。");
                        System.out.println(new String(buffer.array(),0,result));
                    }
                    @Override
                    public void failed(Throwable exc, Integer attachment) {
                    }
                });
            }
            //失败回调
            @Override
            public void failed(Throwable exc, Integer attachment) {
            }
        });
        System.out.println("跑这儿来了。");
        Thread.sleep(10000);

```