### ReentrantLock

[[java并发/_resources/ReentrantLock/86322ec45778fcf3b59dfc7030acfbdf_MD5.jpeg|Open: lock.png]]
![[java并发/_resources/ReentrantLock/86322ec45778fcf3b59dfc7030acfbdf_MD5.jpeg]]

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201006214723561.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2V2ZXJ5X19kYXk=,size_16,color_FFFFFF,t_70#pic_center)
### ReadWritelock

 ==***读 - 读***==  不会阻塞
==***读 - 读 - 写***==  前面读不阻塞 到写 需要等 前面的读 释放锁资源后 才能加锁成功
==***读 -读 - 写 - 读***== ：最后的读需要等写释放后才能加锁成功

[[java并发/_resources/ReentrantLock and ReadWriteLock/58333cb9e5186b9f3e5ac217132f76de_MD5.jpeg|Open: readwritelock.png]]
![[java并发/_resources/ReentrantLock and ReadWriteLock/58333cb9e5186b9f3e5ac217132f76de_MD5.jpeg]]