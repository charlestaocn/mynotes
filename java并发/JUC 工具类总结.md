### CountDownLatch

使用 AQS  的  volatile  state 进行 cas  实现

CountDownLatch :: await 方法将线程 `park` 住
CountDownLatch ::countDown 进行 扣减。。当扣减到 0 时 才 `unpark` 线程

 >因为只能一次减一个 所以需要一次减多个的时候可以自己继承实现
 
 ```java
CountDownLatch countDownLatch = new CountDownLatch(1);  
new Thread(() -> {  
    try {  
        TimeUnit.SECONDS.sleep(10);//等待10秒后执行减state  
    } catch (InterruptedException e) {  
        throw new RuntimeException(e);  
    }  
    countDownLatch.countDown();  
}).start();  
boolean await = countDownLatch.await(5, TimeUnit.SECONDS);//等待5秒,如果5秒内countDownLatch未减完,则返回false，否则为true  
System.out.println("countdownRes:" + await);// countdownRes:false
```
###  CyclicBarrier

使用 ReentrantLock :: Condition 对象实现了  `synchronized` 的 wait 方法

在没有足够线程的情况下将到达的线程 `wait` 住 ，阻塞在  各自的 `await` 方法前, 然后当满足全部后 将他们全部唤醒  并执行 构造时 的方法

> 用于将多个线程集中到一起(一个时间点)执行 或  实现类似 前段 promiseall 的 多线程前置任务都执行结束后执行下一步操作  回调的编程方法

```java
CyclicBarrier cyclicBarrier = new CyclicBarrier(4, () -> {  
    System.out.println(" =========================================");  
    System.out.println(" all task  finished , ok to next  step ...");  
});  
  
IntStream.range(0, 3).forEach((i) -> {  
    new Thread(() -> {  
        try {  
            TimeUnit.SECONDS.sleep(i);  
            System.out.println(Thread.currentThread().getName() + " finished !");  
            cyclicBarrier.await();  
            System.out.println(Thread.currentThread().getName() + "  get awake !");  
        } catch (InterruptedException | BrokenBarrierException e) {  
            throw new RuntimeException(e);  
        }  
    }, "task:" + i).start();  
});  
new Thread(() -> {  
    try {  
        TimeUnit.SECONDS.sleep(4);  
        System.out.println("last task finished ...");  
        cyclicBarrier.await();  
    } catch (InterruptedException | BrokenBarrierException e) {  
        throw new RuntimeException(e);  
    }  
}, "last task").start();


  
//task:0 finished !  
//task:1 finished !  
//task:2 finished !  
//last task finished ...  
// =========================================  
// all task  finished , ok to next  step ...  
//task:0  get awake !  
//task:1  get awake !  
//task:2  get awake !

```


### Semaphore 应用

使用 AQS  的  volatile  state 进行 cas  实现 ，与CountDownLatch类似。 只是多了 对 state 的增加  方法

>  可用于实现 令牌桶之类的流控 
>  缺点是没有实现对 限制的 个数进行set的方法 只能 构建的时候制定 ==需要的可以自己实现==

```java

Semaphore semaphore = new Semaphore(3);

IntStream.range(0, 5).forEach((i) -> {  
    new Thread(() -> {  
        try {  
            System.out.println(Thread.currentThread().getName() + " acquire lock !");  
            semaphore.acquire();  
            System.out.println(Thread.currentThread().getName() + " acquired !");  
            TimeUnit.SECONDS.sleep(i);  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        }finally {  
            semaphore.release();  
            System.out.println(Thread.currentThread().getName() + "  release lock  !");  
        }  
    }, "task:" + i).start();  
});
```