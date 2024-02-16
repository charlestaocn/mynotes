
 > _“Netty是由[JBOSS](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/309533.htm)提供的一个[java开源](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/629119.htm)框架。Netty提供异步的、[事件驱动](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/536048.htm)的网络[应用程序](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/330120.htm)框架和工具，用以快速开发高性能、高可靠性的[网络服务器](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/813.htm)和[客户端](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/930.htm)程序。_ _也就是说，Netty 是一个基于NIO的客户，服务器端编程框架，使用Netty 可以确保你快速和简单的开发出一个网络应用，例如实现了某种协议的客户，[服务端](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/1087294.htm)应用。Netty相当简化和流线化了网络应用的编程开发过程，例如，TCP和UDP的socket服务开发。_ _“快 速”和“简单”并不意味着会让你的最终应用产生维护性或性能上的问题。Netty 是一个吸收了多种协议的实现经验，这些协议包括FTP,SMTP,HTTP，各种二进制，文本协议，并经过相当精心设计的项目，最终，Netty 成功的找到了一种方式，在保证易于开发的同时还保证了其应用的性能，稳定性和伸缩性_。“

-----来自百度百科

## **Netty 简介**

Netty 是对 Java NIO (non blocking io) 的进一步封装和优化，它使用了单线程Event Loop 的模式来避免创建新线程和加锁的开销， 利用了Java NIO 提供的[zero copy buffer](https://link.zhihu.com/?target=https%3A//www.ibm.com/developerworks/cn/linux/l-cn-zerocopy1/%25EF%25BC%258C) 技术来提高cpu 的利用率。当然Java NIO 本身就是对传统IO (oio , old fashion io , blocking io) 技术的一种优化, Java NIO 是利用对IO 多路复用 (epoll 或事 kqueue）从而提高IO的处理效率。

## **Netty 原理**

Netty 主要有三大类： 1 Channel, 2 ChannelPipeline, 3 EventLoop

### **1 Channel**

```text
/**
 * A nexus to a network socket or a component which is capable of I/O
 * operations such as read, write, connect, and bind.
 * <p>
 * A channel provides a user:
   <li>the current state of the channel (e.g. is it open? is it connected?),</li>
 * <li>the {@linkplain ChannelConfig configuration parameters} of the channel (e.g. receive buffer size),</li>
 * <li>the I/O operations that the channel supports (e.g. read, write, connect, and bind), and</li>
 * <li>the {@link ChannelPipeline} which handles all I/O events and requests
 *     associated with the channel.</li>
 * </ul>
```

Channel 等同于Nio 里面的channel, 但是多了很多其他的特性， 例如可以链式监听channel 里面数据的读取和写入。 每个channel 都会与一个EventLoop 相关联。

![[netty/_resources/netty/d65f80228bc49d3100824ce785a5004a_MD5.webp]]

NioServerSocketChannel 就是对Java 中ServerSocketChannel 的一种封装, 主要用于接受发来的socket请求，然后把发来的请求变成 NioSocketChannel 转发到Worker EventloopGroup 里面去处理

### **2 ChannelPipeline**

每个Channel 都会有一个ChannelPipeline 用来把回调函数按顺序链起来。

这个Pipeline 其实是一个double linkedlist 也就是双向链表。里面的addlast， addfirst 会把handler 加到这个链表的的最后面 或是最前面。 当有读写监听的事件发生的时候，这个类会从头部开始一个接着一个接着回调这些监听事件的函数。

```text
// add handler to the head 
@Override
    public final ChannelPipeline addFirst(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler);
            name = filterName(name, handler);

            newCtx = newContext(group, name, handler);

            addFirst0(newCtx);

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                newCtx.setAddPending();
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callHandlerAdded0(newCtx);
                    }
                });
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    }

  private void addFirst0(AbstractChannelHandlerContext newCtx) {
        AbstractChannelHandlerContext nextCtx = head.next;
        newCtx.prev = head;
        newCtx.next = nextCtx;
        head.next = newCtx;
        nextCtx.prev = newCtx;
    }

// 上面的代码会把handler 加入到链表的末尾。

// 当已经 read 一段数据成功后 开始回调head 的函数
 @Override
public final ChannelPipeline fireChannelRead(Object msg) {
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}

static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRead(m);
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRead(m);
                }
            });
        }
    }
//然后执行 channelread
private void invokeChannelRead(Object msg) {
        if (invokeHandler()) {
            try {
                ((ChannelInboundHandler) handler()).channelRead(this, msg);
            } catch (Throwable t) {
                notifyHandlerException(t);
            }
        } else {
            fireChannelRead(msg);
        }
    }
// 然后找到下一个可以读的 handler 也就是inbound handler 继续回调
// 直到结束
@Override
    public ChannelHandlerContext fireChannelRead(final Object msg) {
        invokeChannelRead(findContextInbound(), msg);
        return this;
    }

 private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;
        } while (!ctx.inbound);
        return ctx;
    }
```

### **3 EventLoop 和 EventLoopGroup.**

EventLoop 和 EventLoopGroup 应该是整个netty 的精髓。

==EventLoop到底是什么==

功能主要用来处理Event的执行器，这个执行器符合JDK的执行器标准，同时具备定时执行任务的功能。也可以看做是JDK的 `Executor` 实现，相当于Netty版本线程池的实现。


EventLoopGroup是一个含有多个EventLoop 的集合。每个EventLoop 是一个无限循环的单线程， 它有一个任务对列，会一直无限循环的从任务对列里面取任务执行。

我们构建server主要用的是NioEventLoopGroup, NioEventLoop。 NioEventLoop 在原有的功能基础加上Java NIO Selector 选举Channel 的功能，每个NioEventLoop都有一个Selector, NioEventLoop 会首先处理所有selector keys，也就是发来的socket请求。 如果没有请求进来，然后会去任务对列里面的任务。

```text
public final class NioEventLoop extends SingleThreadEventLoop {
 @Override
    protected void run() {
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        runAllTasks();
                    }
                } else {  
                 ....... 
             }
}
```

由于每个EventLoop 都是单线程的，从而避免加锁的开销。 Java NIO（Non Blocking IO） 中Selector 模式也是一个单线程对IO 的多重复用，所以并不会阻塞这个EventLoop。 EventLoopGroup 里面多个EventLoop同时执行从而达到并行的效果。但是需要注意的就是在提交任务到这个任务对列的时候或事 加入 channel 的回调函数时候，一定要避免blocking IO 的发生， 如果有需要IO的请求尽可能使用Netty客户端去发NIO ，而不是直接用传统的IO， 要么就需要另外开一个线程去执行。这种特点也导致了在使用Netty 的时候会有很多坑，有可能一不小心阻塞了整个EventLoop


下面我们从一个例子来讲述，Netty是如何构建NIO Server的，以及里面具体流程是什么

```text
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     p.addLast(new DiscardServerHandler());
                 }
             });

            // Bind and start to accept incoming connections.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            // In this example, this does not happen, but you can do that to gracefully
            // shut down your server.
            f.channel().closeFuture().sync();
```

从上面的例子来看，创建服务器应用的时候需要两个NioEventLoopGroup 一个是boss group, 一个是worker group. Boss group 主要是用于接受ServerSocketChannel 来的io 请求， 然后再把请求具体执行的回调函数转交给worker group 去执行。Netty 可以同时创建出多个服务应用， 不同的服务应用可以去不同的port 监听。如果只有一个服务应用的话， 最好设置boss group 里面的thread 个数为1， 这样就可以使得性能达到最好的效果。

上面的代码会执行以下的流程：

1. ServerBootstrap 会利用channelfactory 把NioServerSocketChannel 给利用反射的方式创建出来.
2. 然后在对channel 初始化的时候，会在NioServerSocketChannel 的channelPipeline里面 加入了一个ServerBootstrapAcceptor 的回调函数。
3. 然后把NioServerSocketChannel和Boss NioEventLoopGroup 中一个NioEventLoop 绑定起来. 而且每个NioEventLoop 也会有一个Java NIO selector， NioEventLoop 会把NioServerSocketChannel 注册到这个selector 里面，这个selector 会利用epoll 或事 kqueue 进行对IO 的多重复用

```text
   public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
    }


final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // 注册channel 到boss group 里面
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }

 @Override
    void init(Channel channel) throws Exception {
        .......
        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        .....
        加入 ServerBootstrapAcceptor 回调函数
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }



最终会注册到eventloop 里面的selector 里面
@Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
               ....
            }
        }
    }
```

成功注册之后, 当有请求来的时候， 在Channelpipeline上 里面ServerBootstrapAcceptor 的回调函数会被调用。这个acceptor 会把接受到请求传给children group 也就是worker group

```text
private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {

        private final EventLoopGroup childGroup;
        private final ChannelHandler childHandler;
        private final Entry<ChannelOption<?>, Object>[] childOptions;
        private final Entry<AttributeKey<?>, Object>[] childAttrs;
        private final Runnable enableAutoReadTask;

        ServerBootstrapAcceptor(
                final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
                Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
            this.childGroup = childGroup;
            this.childHandler = childHandler;
            this.childOptions = childOptions;
            this.childAttrs = childAttrs；
        }

        @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
          // NioSocketChannel 每一个socket 的连接  
          final Channel child = (Channel) msg;
          // childhandler 是我们自己定义的处理每个请求的回调函数
          // 会在这里被加入到NioSocketChannel 的channelpipeline 里面
            child.pipeline().addLast(childHandler);

           ....
              
            try {
                //把接受到的socketchannel 注册到childgroup 里面 
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }

        private static void forceClose(Channel child, Throwable t) {
            child.unsafe().closeForcibly();
            logger.warn("Failed to register an accepted channel: {}", child, t);
        }

        .....
}
```

workergroup 和boss group 是同一个类， 所以每个nio 请求会在worker group 里面会像boss group 里面重新走一遍， 不同的是Channelpipline 上面的回调函数是真正执行这个请求的函数而不是这个ServerBootstrapAcceptor。 所以说boss 的EventloopGroup 做的主要事情是接受请求，然后把请求分发给worker 的EventloopGroup， 让worker 的EventloopGroup 去。

具体流程图如下：

![[netty/_resources/netty/957ab756137f0435e4e43ffbb6da8c68_MD5.webp]]

  

Netty 性能非常的好，而且即使在超高并发的情况下也不像tomcat 这种占用机器的大量的资源。有人测试过Netty 和Ngnix 的性能，发现他们的性能相差不大， 但是由于Netty 是java 高级语言实现， 所以的相对ngnix + lua 更容易实现定制。

[C, Erlang, Java and Go Web Server performance test​timyang.net/programming/c-erlang-java-performance/![[netty/_resources/netty/3908ce8ab9109b3b9ae82619e1baedcd_MD5.png]]

但是由于Netty 本身是完全异步的，所以用好它并不容易。Netty 的开发对开发者的要求更高，开发者要明白什么时候需要另开线程去避免对eventloop 的阻塞。 各种回调函数交织在一起也会使得代码变得更复杂，难懂，难以维护，尤其是实现胶水逻辑的时候， 复杂的业务逻辑 将会给开发者带来噩梦，即便有vert.x 这样的优秀的框架出现来试图用Reactive Functional Programming 的方式解决这样的问题， 但是相对于tomcat 或jetty 这种一个请求一个thread 的同步式调用还是难了许多。这也是为什么tomcat 或事jetty 这种一直占据这java web 服务器市场的开发。 因为有时候多加几台服务器远远比多招很多高质量的码农更容易，更节约成本。 Netty 更适合用于业务逻辑相对简单，胶水逻辑少，对性能要求高的单点服务，像单个微服务， 或事对性能要求非常高的开源框架例如google rpc， elasticsearch， spark。