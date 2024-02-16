
![[jvm/_resources/runtime data area/9ef7eb043979e23e076c05f89d6fb5a7_MD5.png]]

![[jvm/_resources/runtime data area/620577e9e3e4f769f189833ad04948c7_MD5.webp]]



![[jvm/_resources/runtime data area/6edea9b4f0bd483ff00e851adf743ff4_MD5.jpg]]


### jvm stack 

[[jvm/_resources/runtime data area/bf7e9577a9057ecc34aa6e9cfa470b4f_MD5.jpeg|Open: jvmstack 1.png]]
![[jvm/_resources/runtime data area/bf7e9577a9057ecc34aa6e9cfa470b4f_MD5.jpeg]]


### 方法区

方法区是所有线程共享的，它是在JVM启动的时候创建的。它保存所有被JVM加载的类和接口的运行时常量池，成员变量以及方法的信息，静态变量以及方法的字节码。JVM的提供者可以通过不同的方式来实现方法区。在Oracle 的HotSpot JVM里，方法区被称为永久区或者永久代（PermGen）。是否对方法区进行垃圾回收对JVM的实现是可选的。

==JDK1.8以后，PermGen被永久移除，有Metaspace(元空间)来实现方法区。==

与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用==本地内存==。

==**方法区移至Metaspace，字符串常量移至Java Heap**。==

因此，默认情况下，元空间的大小仅受本地内存限制

![](https://img-blog.csdnimg.cn/20201118100130454.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjM4MzY4MA==,size_16,color_FFFFFF,t_70)![](https://img-blog.csdnimg.cn/20201118100238249.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjM4MzY4MA==,size_16,color_FFFFFF,t_70)![](https://img-blog.csdnimg.cn/20201118100344348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjM4MzY4MA==,size_16,color_FFFFFF,t_70)