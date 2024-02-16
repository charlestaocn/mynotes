同步阻塞式IO，服务器实现模式为一个连接一个线程，即客户端发送请求服务器端就需要启动一个线程处理，若这个连接不做任何事情会造成不必要的线程开销，当然可以通过==线程池机制改善(主线程接收连接，开子线程接收读取数据)==。

```java
 ServerSocket server = new ServerSocket(9090,20);

        System.out.println("step1: new ServerSocket(9090) ");

        while (true) {
            Socket client = server.accept();  //阻塞1
            System.out.println("step2:client\t" + client.getPort());

			//开线程接收读取数据
            new Thread(new Runnable(){
                public void run() {
                    InputStream in = null;
                    try {
                        in = client.getInputStream();
                        BufferedReader reader = new BufferedReader(new InputStreamReader(in));
                        while(true){
                            String dataline = reader.readLine(); //阻塞2

                            if(null != dataline){
                                System.out.println(dataline);
                            }else{
                                client.close();
                                break;
                            }
                        }
                        System.out.println("客户端断开");

                    } catch (IOException e) {
                        e.printStackTrace();
                    }

                }



            }).start();

        }
```

![[bio.png]]


从图中可以看出，传统IO的特点在于：

- 每个客户端连接到达之后，服务端会分配一个线程给该客户端，该线程会处理包括读取数据，解码，业务计算，编码，以及发送数据整个过程；
- 同一时刻，服务端的吞吐量与服务器所提供的线程数量是呈线性关系的。

 这种设计模式在客户端连接不多，并发量不大的情况下是可以运行得很好的，但是在==海量并发的情况下，这种模式就显得力不从心了==。这种模式主要存在的问题有如下几点：

- 服务器的并发量对服务端能够创建的线程数有很大的依赖关系，但是服务器线程却是不能无限增长的；
- 服务端每个线程不仅要进行IO读写操作，而且还需要进行业务计算；
- 服务端在获取客户端连接，读取数据，以及写入数据的过程都是阻塞类型的，在网络状况不好的情况下，这将极大的降低服务器每个线程的利用率，从而降低服务器吞吐量。
