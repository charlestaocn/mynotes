
## 安装

可以在官方 Github 上进行下载，如果速度较慢，可以尝试国内的码云 Gitee 下载。

```text
curl -O https://arthas.aliyun.com/arthas-boot.jar

```

## 运行

**Arthas** 只是一个 java 程序，所以可以直接用 `java -jar` 运行。运行时或者运行之后要选择要监测的 Java 进程。

```powershell
# 运行方式1，先运行，在选择 Java 进程 PID
java -jar arthas-boot.jar
# 选择进程(输入[]内编号(不是PID)回车)
[INFO] arthas-boot version: 3.1.4
[INFO] Found existing java process, please choose one and hit RETURN.
* [1]: 11616 com.Arthas
  [2]: 8676
  [3]: 16200 org.jetbrains.jps.cmdline.Launcher
  [4]: 21032 org.jetbrains.idea.maven.server.RemoteMavenServer

# 运行方式2，运行时选择 Java 进程 PID
java -jar arthas-boot.jar [PID]
```


查看 PID 的方式可以通过 `ps` 命令，也可以通过 JDK 提供的 `jps`命令。

```text
# 查看运行的 java 进程信息
$ jps -mlvV 
# 筛选 java 进程信息
$ jps -mlvV | grep [xxx]
```

`jps` 筛选想要的进程方式。

  

![[jvm/_resources/arthas/6f781b8279106f18e99912d85cc68423_MD5.webp]]

在出现 **Arthas** Logo 之后就可以使用命令进行问题诊断了。下面会详细介绍。

![[jvm/_resources/arthas/559d43e3ce3194bc2bce51750d488eb6_MD5.webp]]

  

更多的启动方式可以参考 help 帮助命令。

```powershell
# 其他用法
EXAMPLES:
  java -jar arthas-boot.jar <pid>
  java -jar arthas-boot.jar --target-ip 0.0.0.0
  java -jar arthas-boot.jar --telnet-port 9999 --http-port -1
  java -jar arthas-boot.jar --tunnel-server 'ws://192.168.10.11:7777/ws'
  java -jar arthas-boot.jar --tunnel-server 'ws://192.168.10.11:7777/ws'
--agent-id bvDOe8XbTM2pQWjF4cfw
  java -jar arthas-boot.jar --stat-url 'http://192.168.10.11:8080/api/stat'
  java -jar arthas-boot.jar -c 'sysprop; thread' <pid>
  java -jar arthas-boot.jar -f batch.as <pid>
  java -jar arthas-boot.jar --use-version 3.1.4
  java -jar arthas-boot.jar --versions
  java -jar arthas-boot.jar --session-timeout 3600
  java -jar arthas-boot.jar --attach-only
  java -jar arthas-boot.jar --repo-mirror aliyun --use-http
```

## 3.3 web console

**Arthas** 目前支持 `Web Console`，在成功启动连接进程之后就已经自动启动，可以直接访问 [http://127.0.0.1:8563/](https://link.zhihu.com/?target=http%3A//127.0.0.1%3A8563/) 访问，页面上的操作模式和控制台完全一样。

![[jvm/_resources/arthas/2ad40096faa770fd120195238f135b07_MD5.webp]]

## 3.4 常用命令

下面列举一些 **[Arthas](https://link.zhihu.com/?target=https%3A//www.wdbyte.com/2019/11/arthas/)** 的常用命令，看到这里你可能还不知道怎么使用，别急，后面会一一介绍。

## 3.5 退出

使用 shutdown 退出时 **Arthas** 同时自动重置所有增强过的类 。

## 4、Arthas 常用操作

上面已经了解了什么是 **Arthas**，以及 **Arthas** 的启动方式，下面会依据一些情况，详细说一说 **Arthas** 的使用方式。在使用命令的过程中如果有问题，每个命令都可以是 `-h` 查看帮助信息。

首先编写一个有各种情况的测试类运行起来，再使用 **Arthas** 进行问题定位，

```java
import java.util.HashSet;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import lombok.extern.slf4j.Slf4j;

/**
 * <p>
 * Arthas Demo
 * 公众号：未读代码
 *
 * @Author niujinpeng
 */
@Slf4j
public class Arthas {

    private static HashSet hashSet = new HashSet();
    /** 线程池，大小1*/
    private static ExecutorService executorService = Executors.newFixedThreadPool(1);

    public static void main(String[] args) {
        // 模拟 CPU 过高，这里注释掉了，测试时可以打开
        // cpu();
        // 模拟线程阻塞
        thread();
        // 模拟线程死锁
        deadThread();
        // 不断的向 hashSet 集合增加数据
        addHashSetThread();
    }

    /**
     * 不断的向 hashSet 集合添加数据
     */
    public static void addHashSetThread() {
        // 初始化常量
        new Thread(() -> {
            int count = 0;
            while (true) {
                try {
                    hashSet.add("count" + count);
                    Thread.sleep(10000);
                    count++;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }

    public static void cpu() {
        cpuHigh();
        cpuNormal();
    }

    /**
     * 极度消耗CPU的线程
     */
    private static void cpuHigh() {
        Thread thread = new Thread(() -> {
            while (true) {
                log.info("cpu start 100");
            }
        });
        // 添加到线程
        executorService.submit(thread);
    }

    /**
     * 普通消耗CPU的线程
     */
    private static void cpuNormal() {
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                while (true) {
                    log.info("cpu start");
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }

    /**
     * 模拟线程阻塞,向已经满了的线程池提交线程
     */
    private static void thread() {
        Thread thread = new Thread(() -> {
            while (true) {
                log.debug("thread start");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        // 添加到线程
        executorService.submit(thread);
    }

    /**
     * 死锁
     */
    private static void deadThread() {
        /** 创建资源 */
        Object resourceA = new Object();
        Object resourceB = new Object();
        // 创建线程
        Thread threadA = new Thread(() -> {
            synchronized (resourceA) {
                log.info(Thread.currentThread() + " get ResourceA");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info(Thread.currentThread() + "waiting get resourceB");
                synchronized (resourceB) {
                    log.info(Thread.currentThread() + " get resourceB");
                }
            }
        });

        Thread threadB = new Thread(() -> {
            synchronized (resourceB) {
                log.info(Thread.currentThread() + " get ResourceB");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info(Thread.currentThread() + "waiting get resourceA");
                synchronized (resourceA) {
                    log.info(Thread.currentThread() + " get resourceA");
                }
            }
        });
        threadA.start();
        threadB.start();
    }
}
```

## 4.1 全局监控

使用 **dashboard** 命令可以概览程序的 线程、内存、GC、运行环境信息。

![[jvm/_resources/arthas/1965634ca6dd36820d2297e9ec138152_MD5.webp]]

## 4.2 CPU 为什么起飞了

上面的代码例子有一个 `CPU` 空转的死循环，非常的消耗 `CPU性能`，那么怎么找出来呢？

使用 **thread**查看**所有**线程信息，同时会列出每个线程的 `CPU` 使用率，可以看到图里 ID 为12 的线程 CPU 使用100%。  

![[jvm/_resources/arthas/dccd24e7dcd089635f4cf5c16f676401_MD5.png]]

使用命令 **thread 12** 查看 CPU 消耗较高的 12 号线程信息，可以看到 CPU 使用较高的方法和行数（这里的行数可能和上面代码里的行数有区别，因为上面的代码在我写文章时候重新排过版了）。

![[jvm/_resources/arthas/1e6936efcc20babe3d843ae993fb63ac_MD5.webp]]

上面是先通过观察总体的线程信息，然后查看具体的线程运行情况。如果只是为了寻找 CPU 使用较高的线程，可以直接使用命令 **thread -n [显示的线程个数]** ，就可以排列出 CPU 使用率 **Top N** 的线程。

![[jvm/_resources/arthas/e1a5265fd4a33c91ba58e33875089196_MD5.webp]]

定位到的 CPU 使用最高的方法。

![[jvm/_resources/arthas/0895e7bfe8636fb290b301be2ff88fa0_MD5.webp]]

  

## 4.3 线程池线程状态

定位线程问题之前，先回顾一下线程的几种常见状态：

- **RUNNABLE** 运行中
- **TIMED_WAITIN** 调用了以下方法的线程会进入**TIMED_WAITING**：

1. Thread#sleep()
2. Object#wait() 并加了超时参数
3. Thread#join() 并加了超时参数
4. LockSupport#parkNanos()
5. LockSupport#parkUntil()

  

- **WAITING** 当线程调用以下方法时会进入WAITING状态：

1. Object#wait() 而且不加超时参数
2. Thread#join() 而且不加超时参数
3. LockSupport#park()

  

- **BLOCKED** 阻塞，等待锁

上面的模拟代码里，定义了线程池大小为1 的线程池，然后在 `cpuHigh` 方法里提交了一个线程，在 `thread`方法再次提交了一个线程，后面的这个线程因为线程池已满，会阻塞下来。

使用 **thread | grep pool** 命令查看线程池里线程信息。

![[jvm/_resources/arthas/80c3fe6e077a992625ada2c6e55b0925_MD5.png]]

可以看到线程池有 **WAITING** 的线程。

![[jvm/_resources/arthas/57f63d0b7d6ef7faf03154f2bc9c78af_MD5.webp]]

  

## 4.4 线程死锁

上面的模拟代码里 `deadThread` 方法实现了一个死锁，使用 **thread -b** 命令查看直接定位到死锁信息。

```java
/**
 * 死锁
 */
private static void deadThread() {
    /** 创建资源 */
    Object resourceA = new Object();
    Object resourceB = new Object();
    // 创建线程
    Thread threadA = new Thread(() -> {
        synchronized (resourceA) {
            log.info(Thread.currentThread() + " get ResourceA");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.info(Thread.currentThread() + "waiting get resourceB");
            synchronized (resourceB) {
                log.info(Thread.currentThread() + " get resourceB");
            }
        }
    });

    Thread threadB = new Thread(() -> {
        synchronized (resourceB) {
            log.info(Thread.currentThread() + " get ResourceB");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.info(Thread.currentThread() + "waiting get resourceA");
            synchronized (resourceA) {
                log.info(Thread.currentThread() + " get resourceA");
            }
        }
    });
    threadA.start();
    threadB.start();
}
```

检查到的死锁信息。

![[jvm/_resources/arthas/57772f058b57b6eb9b89bf97a51d0d32_MD5.webp]]

  

## 4.5 反编译

上面的代码放到了包 `com`下，假设这是一个线程环境，当怀疑当前运行的代码不是自己想要的代码时，可以直接反编译出代码，也可以选择性的查看类的字段或方法信息。

如果怀疑不是自己的代码，可以使用 **jad** 命令直接反编译 class。

![[jvm/_resources/arthas/4457a8ffdd6cc927e3d5d4381cec82b4_MD5.webp]]

`jad` 命令还提供了一些其他参数：

```text
# 反编译只显示源码
jad --source-only com.Arthas
# 反编译某个类的某个方法
jad --source-only com.Arthas mysql
```

## 4.6 查看字段信息

使用 **sc -d -f ** 命令查看类的字段信息。

```powershell
[arthas@20252]$ sc -d -f com.Arthas
sc -d -f com.Arthas
 class-info        com.Arthas
 code-source       /C:/Users/Niu/Desktop/arthas/target/classes/
 name              com.Arthas
 isInterface       false
 isAnnotation      false
 isEnum            false
 isAnonymousClass  false
 isArray           false
 isLocalClass      false
 isMemberClass     false
 isPrimitive       false
 isSynthetic       false
 simple-name       Arthas
 modifier          public
 annotation
 interfaces
 super-class       +-java.lang.Object
 class-loader      +-sun.misc.Launcher$AppClassLoader@18b4aac2
                     +-sun.misc.Launcher$ExtClassLoader@2ef1e4fa
 classLoaderHash   18b4aac2
 fields            modifierfinal,private,static
                   type    org.slf4j.Logger
                   name    log
                   value   Logger[com.Arthas]

                   modifierprivate,static
                   type    java.util.HashSet
                   name    hashSet
                   value   [count1, count2]

                   modifierprivate,static
                   type    java.util.concurrent.ExecutorService
                   name    executorService
                   value   java.util.concurrent.ThreadPoolExecutor@71c03156[Ru
                           nning, pool size = 1, active threads = 1, queued ta
                           sks = 0, completed tasks = 0]


Affect(row-cnt:1) cost in 9 ms.
```

## 4.7 查看方法信息

使用 **sm** 命令查看类的方法信息。

```text
[arthas@22180]$ sm com.Arthas
com.Arthas <init>()V
com.Arthas start()V
com.Arthas thread()V
com.Arthas deadThread()V
com.Arthas lambda$cpuHigh$1()V
com.Arthas cpuHigh()V
com.Arthas lambda$thread$3()V
com.Arthas addHashSetThread()V
com.Arthas cpuNormal()V
com.Arthas cpu()V
com.Arthas lambda$addHashSetThread$0()V
com.Arthas lambda$deadThread$4(Ljava/lang/Object;Ljava/lang/Object;)V
com.Arthas lambda$deadThread$5(Ljava/lang/Object;Ljava/lang/Object;)V
com.Arthas lambda$cpuNormal$2()V
Affect(row-cnt:16) cost in 6 ms.
```

## 4.8 对变量的值很是好奇

使用 **ognl** 命令，ognl 表达式可以轻松操作想要的信息。

代码还是上面的示例代码，我们查看变量 `hashSet` 中的数据：

![[jvm/_resources/arthas/fba064ee0f8bf879c325974bd2d1b38f_MD5.webp]]

  

查看静态变量 `hashSet` 信息。

```powershell
[arthas@19856]$ ognl '@com.Arthas@hashSet'
@HashSet[
    @String[count1],
    @String[count2],
    @String[count29],
    @String[count28],
    @String[count0],
    @String[count27],
    @String[count5],
    @String[count26],
    @String[count6],
    @String[count25],
    @String[count3],
    @String[count24],
```

查看静态变量 hashSet 大小。

```text
[arthas@19856]$ ognl '@com.Arthas@hashSet.size()'
	@Integer[57]
```

甚至可以进行操作。

```powershell
[arthas@19856]$ ognl  '@com.Arthas@hashSet.add("test")'
	@Boolean[true]
[arthas@19856]$
# 查看添加的字符
[arthas@19856]$ ognl  '@com.Arthas@hashSet' | grep test
    @String[test],
[arthas@19856]$
```

`ognl` 可以做很多事情，可以参考 [ognl 表达式特殊用法( https://github.com/alibaba/arthas/issues/71 )](https://link.zhihu.com/?target=https%3A//github.com/alibaba/arthas/issues/71)。

## 4.9 程序有没有问题

### 4.9.1 运行较慢、耗时较长

使用 **trace** 命令可以跟踪统计方法耗时

这次换一个模拟代码。一个最基础的 Springboot 项目（当然，不想 Springboot 的话，你也可以直接在 UserController 里 main 方法启动）控制层 `getUser` 方法调用了 `userService.get(uid);`，这个方法中分别进行`check`、`service`、`redis`、`mysql`操作。

```java
@RestController
@Slf4j
public class UserController {

    @Autowired
    private UserServiceImpl userService;

    @GetMapping(value = "/user")
    public HashMap<String, Object> getUser(Integer uid) throws Exception {
        // 模拟用户查询
        userService.get(uid);
        HashMap<String, Object> hashMap = new HashMap<>();
        hashMap.put("uid", uid);
        hashMap.put("name", "name" + uid);
        return hashMap;
    }
}
```

模拟代码 Service:

```java
@Service
@Slf4j
public class UserServiceImpl {

    public void get(Integer uid) throws Exception {
        check(uid);
        service(uid);
        redis(uid);
        mysql(uid);
    }

    public void service(Integer uid) throws Exception {
        int count = 0;
        for (int i = 0; i < 10; i++) {
            count += i;
        }
        log.info("service  end {}", count);
    }

    public void redis(Integer uid) throws Exception {
        int count = 0;
        for (int i = 0; i < 10000; i++) {
            count += i;
        }
        log.info("redis  end {}", count);
    }

    public void mysql(Integer uid) throws Exception {
        long count = 0;
        for (int i = 0; i < 10000000; i++) {
            count += i;
        }
        log.info("mysql end {}", count);
    }

 	 public boolean check(Integer uid) throws Exception {
         if (uid == null || uid < 0) {
             log.error("uid不正确，uid:{}", uid);
             throw new Exception("uid不正确");
         }
         return true;
     }
}
```

运行 Springboot 之后，使用 **trace== ** 命令开始检测耗时情况。

```text
[arthas@6592]$ trace com.UserController getUser
```

访问接口 `/getUser` ，可以看到耗时信息，看到 `com.UserServiceImpl:get()` 方法耗时较高。  

![[jvm/_resources/arthas/6202370cf0e51b7d2ec7baa74c483b26_MD5.webp]]

继续跟踪耗时高的方法，然后再次访问。

```text
[arthas@6592]$ trace com.UserServiceImpl get
```

![[jvm/_resources/arthas/ced8cfec57b2ded48c026e66f02bacb4_MD5.webp]]

很清楚的看到是 `com.UserServiceImpl` 的 `mysql` 方法耗时是最高的。

```java
Affect(class-cnt:1 , method-cnt:1) cost in 31 ms.
`---ts=2019-10-16 14:40:10;thread_name=http-nio-8080-exec-8;id=1f;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@23a918c7
    `---[6.792201ms] com.UserServiceImpl:get()
        +---[0.008ms] com.UserServiceImpl:check() #17
        +---[0.076ms] com.UserServiceImpl:service() #18
        +---[0.1089ms] com.UserServiceImpl:redis() #19
        `---[6.528899ms] com.UserServiceImpl:mysql() #20
```

### 4.9.2 统计方法耗时

使用 **monitor** 命令监控统计方法的执行情况。

每5秒统计一次 `com.UserServiceImpl` 类的 `get` 方法执行情况。

```text
monitor -c 5 com.UserServiceImpl get
```

![[jvm/_resources/arthas/85038d39f8d7d9a6d00eab064563ea96_MD5.webp]]

  

## 4.10 想观察方法信息

下面的示例用到了文章的前两个模拟代码。

### 4.10.1 观察方法的入参出参信息

使用 **watch** 命令轻松查看输入输出参数以及异常等信息。

```powershell
USAGE:
   watch [-b] [-e] [-x <value>] [-f] [-h] [-n <value>] [-E] [-M <value>] [-s] class-pattern method-pattern express [condition-express]

 SUMMARY:
   Display the input/output parameter, return object, and thrown exception of specified method invocation
   The express may be one of the following expression (evaluated dynamically):
           target : the object
            clazz : the object's class
           method : the constructor or method
           params : the parameters array of method
     params[0..n] : the element of parameters array
        returnObj : the returned object of method
         throwExp : the throw exception of method
         isReturn : the method ended by return
          isThrow : the method ended by throwing exception
            #cost : the execution time in ms of method invocation
 Examples:
   watch -b org.apache.commons.lang.StringUtils isBlank params
   watch -f org.apache.commons.lang.StringUtils isBlank returnObj
   watch org.apache.commons.lang.StringUtils isBlank '{params, target, returnObj}' -x 2
   watch -bf *StringUtils isBlank params
   watch *StringUtils isBlank params[0]
   watch *StringUtils isBlank params[0] params[0].length==1
   watch *StringUtils isBlank params '#cost>100'
   watch -E -b org\.apache\.commons\.lang\.StringUtils isBlank params[0]

 WIKI:
   https://alibaba.github.io/arthas/watch
```

常用操作：

```text
# 查看入参和出参
$ watch com.Arthas addHashSet '{params[0],returnObj}'
# 查看入参和出参大小
$ watch com.Arthas addHashSet '{params[0],returnObj.size}'
# 查看入参和出参中是否包含 'count10'
$ watch com.Arthas addHashSet '{params[0],returnObj.contains("count10")}'
# 查看入参和出参，出参 toString
$ watch com.Arthas addHashSet '{params[0],returnObj.toString()}'
```

查看入参出参。

![[jvm/_resources/arthas/63162db5e797539db97839c4cf0387b3_MD5.webp]]

  

查看返回的异常信息。

### 4.10.2 观察方法的调用路径

使用 **stack**命令查看方法的调用信息。

```text
# 观察 类com.UserServiceImpl的 mysql 方法调用路径
stack com.UserServiceImpl mysql
```

  

![[jvm/_resources/arthas/af64cb4cbb40ee8a089540a3f241ee7c_MD5.webp]]

### 4.10.3 方法调用时空隧道

使用 **tt** 命令记录方法执行的详细情况。

> **tt** 命令方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测 。

常用操作：

开始记录方法调用信息：tt -t com.UserServiceImpl check

![[jvm/_resources/arthas/d13d007a62ccab19365d760cbd52c126_MD5.png]]

可以看到记录中 INDEX=1001 的记录的 IS-EXP = true ，说明这次调用出现异常。

查看记录的方法调用信息： tt -l

![[jvm/_resources/arthas/1b67b606da4d5f34fa255be38e9b96f2_MD5.png]]

查看调用记录的详细信息（-i 指定 INDEX）： tt -i 1001

![[jvm/_resources/arthas/6891d85edaa21be8f2c38a04cc5ef1e2_MD5.webp]]

可以看到 INDEX=1001 的记录的异常信息。

重新发起调用，使用指定记录，使用 -p 重新调用。

```text
tt -i 1001 -p
```

![[jvm/_resources/arthas/8918402931d76fd966c66415602d8055_MD5.webp]]

## 4.11. 火焰图分析

最近 Arthas 性能分析工具上线了**火焰图**分析功能，Arthas 使用 **async-profiler** 生成 CPU/内存火焰图进行性能分析，弥补了之前内存分析的不足。在 Arthas 上使用还是比较方便的。

**`profiler`** 命令支持生成应用热点的火焰图。本质上是通过不断的采样，然后把收集到的采样结果生成火焰图。

**`profiler`** 命令基本运行结构是 **`profiler action [actionArg]`**

### **4.11.1.使用案例**

**开启 prifilter**

默认情况下，生成的是cpu的火焰图，即event为`cpu`。可以用`--event`参数来指定，使用 start 命令开始捕获信息。

```text
$ profiler start
Started [cpu] profiling
```

获取已采集的sample的数量

```text
$ profiler getSamples
23
```

查看 profiler状态，可以查看当前 profiler 在采样哪种 `event` 和进行的采样时间。

$ profiler status

[cpu] profiling is running for 4 seconds

**停止profiler**

生成svg格式火焰图

```text
$ profiler stop
profiler output file: /tmp/demo/arthas-output/20191125-135546.svg
OK
```

默认情况下，生成的结果保存到应用的`工作目录`下的`arthas-output`目录。可以通过 `--file`参数来指定输出结果路径。

比如：

```text
$ profiler stop --file /tmp/output.svg
```

**HTML 格式输出**

默认情况下，结果文件是`svg`格式，如果想生成`html`格式，可以用`--format`参数指定：$ profiler stop --format html

**查看 profilter**

默认情况下，arthas使用3658端口，则可以打开： [http://localhost:3658/arthas-output/](https://link.zhihu.com/?target=http%3A//localhost%3A3658/arthas-output/) 查看到`arthas-output`目录下面的profiler结果：

![[jvm/_resources/arthas/258f6954831e7dec304718be31baa143_MD5.webp]]

点击可以查看具体的结果：**火焰图里，横条越长，代表使用的越多，从下到上是调用堆栈信息**

![[jvm/_resources/arthas/280c4ecb88fde065ecffe203b8f254b2_MD5.webp]]

**profilter 自持多种分析方式，**常见的有 event: cpu|alloc|lock|cache-misses etc. 比如要分析内存使用情况。

$ profiler start --event alloc

### 4.11.2. 复杂命令

比如开始采样：

```text
profiler execute 'start'
```

停止采样，并保存到指定文件里：

```text
profiler execute 'stop,file=/tmp/result.svg'
```

## 4.12  热更新代码

### 示例

在 arthas-demo 示例中，一共有两个类，一个 HelloService 类，sayHello 方法负责不断的打印 `hello world`：

```javascript
public class HelloService {

    public void sayHello() {
        System.out.println("hello world");
    }

}
```


HelloService 用于模拟我们日常开发的一些业务 Service，另外还有一个 Main 函数，负责启动进程，并循环调用

```javascript
public class Main {

    public static void main(String[] args) throws InterruptedException {
        HelloService helloService = new HelloService();
        while (true) {
            Thread.sleep(1000);
            helloService.sayHello();
        }
    }

}
```


### 需求

假设这段代码运行在线上，我们希望通过 Arthas 将 `hello world` 的输出更改为 `hello arthas`。

Arthas 修改热更的逻辑主要分为三步：

- jad 命令反编译出内存中的字节码，生成 class 文件
- 修改代码，使用 mc 命令内存编译新的 class 文件
- redefine 重新加载新的 class 文件

从而达到热更新的效果

### jad 反编译

当挂载上 Arthas 之后，执行

```javascript
$ jad --source-only moe.cnkirito.arthas.demo.HelloService > /tmp/HelloService.java
```


将字节码文件输出到指定的位置，查看其内容，与示例中的源码内容一致：

```javascript
/*
 * Decompiled with CFR.
 */
package moe.cnkirito.arthas.demo;

import java.io.PrintStream;

public class HelloService {
    public void sayHello() {
        System.out.println("hello world");
    }
}
```


命令中 `--source-only` 的含义为，只输出源码部分，如果不加这个参数，在反编译出的内容头部会携带类加载器的信息：

```javascript
ClassLoader:
+-sun.misc.Launcher$AppClassLoader@18b4aac2
  +-sun.misc.Launcher$ExtClassLoader@20d5ad12

Location:
/Users/xujingfeng/IdeaProjects/arthas-demo/target/classes/
```


在[服务器](https://cloud.tencent.com/act/pro/promotion-cvm?from_column=20065&from=20065)上可以直接使用 vi 等编辑器对源码进行编辑。将 `hello world` 改为 `hello arthas`，为下一步做准备。

### sc 查找类加载器

mc 命令编译文件需要传入该类对应类加载器的 hash 值，需要先使用 sc 命令查看 HelloService 的累加器信息

```javascript
$ sc -d moe.cnkirito.arthas.demo.HelloService
```


输出：

```javascript
class-info        moe.cnkirito.arthas.demo.HelloService
 code-source       /Users/xujingfeng/IdeaProjects/arthas-demo/target/classes/
 name              moe.cnkirito.arthas.demo.HelloService
 isInterface       false
 isAnnotation      false
 isEnum            false
 isAnonymousClass  false
 isArray           false
 isLocalClass      false
 isMemberClass     false
 isPrimitive       false
 isSynthetic       false
 simple-name       HelloService
 modifier          public
 annotation
 interfaces
 super-class       +-java.lang.Object
 class-loader      +-sun.misc.Launcher$AppClassLoader@18b4aac2
                     +-sun.misc.Launcher$ExtClassLoader@20d5ad12
 classLoaderHash   18b4aac2
```

最后一行 `classLoaderHash` 即为 HelloService 的类加载器 hash 值。

> Arthas 支持 grep，你也可以简化该操作为： sc -d moe.cnkirito.arthas.demo.HelloService | grep classLoaderHash

### mc 内存编译

```javascript
$ mc -c 18b4aac2 /tmp/HelloService.java -d /tmp
Memory compiler output:
/tmp/moe/cnkirito/arthas/demo/HelloService.class
```


使用 `-c` 指定类加载器的 hash 值。编译完成后，/tmp 目录下会生成对应的 class 字节码文件

### redefine 热更新代码

```javascript
$ redefine /tmp/moe/cnkirito/arthas/demo/HelloService.class
```


### 检查结果

```javascript
hello world
hello world
hello world
hello world
hello arthas
hello arthas
hello arthas
hello arthas
```


热更新成功

### 常见问题

#### redefine 使用限制

- 不允许新增或者删除 field/method 会出现类似下面的提示 `redefine error! java.lang.UnsupportedOperationException: class redefinition failed: attempted to change the schema (add/remove fields)`
- 运行中的方法不会立刻生效，会在下一次进入该方法时才能生效。 很好理解，并发问题

#### mc 常见问题

- mc 命令有可能失败 因为运行时环境和编译时环境的 JDK 可能有版本差异，mc 可能会失败。如果编译失败可以在本地编译好 `.class` 文件，再上传到服务器
- 当存在内部类时，一次会生成多个 class 文件 `public` `class` `HelloService` `{` `public` `void` `sayHello()` `{` `Inner.test();` `}` `public` `static` `class` `Inner` `{` `public` `static` `void` `test()` `{` `System.out.println("hello inner");` `}` `}` `}` 执行 mc `$ mc -c 18b4aac2 /tmp/HelloService.java -d /tmp` `Memory compiler output:` `/tmp/moe/cnkirito/arthas/demo/HelloService$Inner.class` `/tmp/moe/cnkirito/arthas/demo/HelloService.class` 注意 redefine 时也可以同时传入多个入参 `$ redefine /tmp/moe/cnkirito/arthas/demo/HelloService$Inner.class` `/tmp/moe/cnkirito/arthas/demo/HelloService.class` `redefine success, size: 2`