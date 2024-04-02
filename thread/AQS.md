# 问题
- 什么是AQS? 为什么它是核心?
- AQS的核心思想是什么? 它是怎么实现的? 底层数据结构等
- AQS有哪些核心的方法?
- AQS定义什么样的资源获取方式? AQS定义了两种资源获取方式：`独占`(只有一个线程能访问执行，又根据是否按队列的顺序分为`公平锁`和`非公平锁`，如`ReentrantLock`) 和`共享`(多个线程可同时访问执行，如`Semaphore`、`CountDownLatch`、 `CyclicBarrier` )。`ReentrantReadWriteLock`可以看成是组合式，允许多个线程同时对某一资源进行读。
- AQS底层使用了什么样的设计模式? 模板
- AQS的应用示例?


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210813223411.png)


![image-20210813182227612](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210813182227.png)


![image-20210813182646488](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210813182646.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210813222326.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210813222638.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210813222710.png)


加锁 lock() ----->线程阻塞
wait()
sleep()
park
while(true)


volatile、CAS、LockSupport.park
公平锁与非公平锁
独占锁、共享锁









# 参考

https://pdai.tech/md/java/thread/java-thread-x-lock-AbstractQueuedSynchronizer.html#%E7%B1%BB%E7%9A%84%E6%A0%B8%E5%BF%83%E6%96%B9%E6%B3%95---release%E6%96%B9%E6%B3%95


https://github.com/cosen1024/Java-Interview/blob/main/Java%E5%B9%B6%E5%8F%91/AQS.md


https://zhuanlan.zhihu.com/p/86072774



https://blog.csdn.net/qq_35190492/article/details/115339297



https://juejin.cn/post/6844903938202796045#heading-0



https://www.cnblogs.com/chengxiao/p/7141160.html#a3



# 面试题

https://cloud.tencent.com/developer/article/1683520

https://juejin.cn/post/6844903873329496077



