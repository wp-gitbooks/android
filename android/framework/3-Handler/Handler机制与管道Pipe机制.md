在深入系统的学习Handler的时候，我们接触到了Looper之所以死循环不会导致CPU使用率过高，是因为使用了Linux下的pipe和epoll机制。


# 一、管道概述 [[0-pipe管道]]

管道，其本质是也是文件，但又和普通的文件会有所不同：管道缓冲区大小一般为1页，即4K字节。管道分为读端和写端，读端负责从管道拿数据，当数据为空时则阻塞；写端向管道写数据，当管道缓存区满时则阻塞。

在Handler机制中，Looper.loop方法会不断循环处理Message，其中消息的获取是通过 Message msg = queue.next(); 方法获取下一条消息。该方法中会调用nativePollOnce()方法，这便是一个native方法，再通过JNI调用进入Native层，在Native层的代码中便采用了管道机制。


# 二、Handler为何使用管道?

我们可能会好奇，既然是同一个进程间的线程通信，为何需要管道呢？

我们知道线程之间内存共享，通过Handler通信，消息池的内容并不需要从一个线程拷贝到另一个线程，因为两线程可使用的内存时同一个区域，都有权直接访问，当然也存在线程私有区域ThreadLocal（这里不涉及）。即然不需要拷贝内存，那管道是何作用呢？

**Handler机制中管道作用**就是当一个线程A准备好Message，并放入消息池，这时需要通知另一个线程B去处理这个消息。线程A向管道的写端写入数据1（对于老的Android版本是写入字符`W`），管道有数据便会唤醒线程B去处理消息。管道主要工作是用于通知另一个线程的，这便是最核心的作用。

这里我们通过两张图来展示Handler在Java层和在Native层的逻辑：

Java层：
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210721115052.png)


Native层：

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210721115133.png)



# 三、Handler为何采用管道而非Binder？

handler不采用Binder，并非binder完成不了这个功能，而是太浪费CPU和内存资源了。因为Binder采用C/S架构，一般用于不同进程间的通信。

- **从内存角度**：通信过程中Binder还涉及一次内存拷贝，handler机制中的Message根本不需要拷贝，本身就是在同一个内存。Handler需要的仅仅是告诉另一个线程数据有了。

- **从CPU角度**，为了Binder通信底层驱动还需要为何一个binder线程池，每次通信涉及binder线程的创建和内存分配等比较浪费CPU资源。

从上面的角度分析可得，Binder用于进程间通信，而Handler消息机制用于同进程的线程间通信，Handler不宜采用Binder。



# 四、Android 6.0及以后的机制

在Android 6.0及以前的版本使用管道与epoll来完成Looper的休眠与唤醒的。

在Android 6.0及以后的新版本中使用的是eventfd与epoll来完成Looper的休眠与唤醒的。

感兴趣的可以进一步的学习和了解管道的知识及eventfd的知识，并比较一下两种机制的优劣，进而明白Android官方为何对此机制进行调整。


# Handler运行机制

MessageQueue的底层实现是利用管道和epoll机制来实现的。



![pipe.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210721125059)

 MessageQueue 在构造方法中，会调用 native 方法 nativeInit 方法，在NativeMessageQueue 的构造方法中，会构造一个 JNI 层的 Looper。Looper.loop()有个for死循环，它调用了MessageQueue下的next方法。

![looper1.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210721125108)





![looper2.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210721125114)

 如上图，nextPollTimeoutMillis这个变量，这个变量代表MessageQueue下次被唤醒的时间。MessageQueue里Message在加入队列的时候，会按照执行的时间顺序排列；每次消息入队列时，MessageQueue都会尽量计算出一个精确的时间。假如这个时间是计算出来是1000ms，如果消息队列中没有消息需要马上处理时，会判断用户是否设置了Idle Handler。如果有的话，则会尝试处理mIdleHandlers中所记录的所有Idle Handler。此时会逐个调用这些Idle Handler的queueIdle()成员函数，再次调用nativePollOnce()方法，线程阻塞住，不占用资源。当时间到了，会往管道流中写入字节流，唤醒线程，处理Message。 ####Looper队列的阻塞唤醒的功能是怎么实现的？ MessageQueue 是按照消息触发时间的先后顺序排列的，队列头部的消息是最早触发的。当有消息加入，会从队列头部开始遍历，插入到合适的位置，以保证所有消息的时间顺序。如果当前线程处于空闲等待状态，需要调用 nativeWake 来唤醒。



```
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jobject obj, jint ptr) {
    // ptr 获取 NativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);  
    return nativeMessageQueue->wake();  
}
复制代码
```

唤醒请求转发到 Looper wake，往管道写入内容，从而唤醒线程。

```
void Looper::wake() {  
    ......  
  
    ssize_t nWrite;  
    do {
        nWrite = write(mWakeWritePipeFd, "W", 1);  // 先管道中写入 "W
    } while (nWrite == -1 && errno == EINTR);  
  
    .......  
}
复制代码
```

当消息队列中没有消息处理时，线程会进入空闲等待状态，具体是通过 Looper 调用 epoll_wait。 附图，

![wake.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210721125123)



- 总结：

   线程在进入循环之前，会在 JNI 创建管道，当消息队列为空时，线程处于空闲等待状态。通过 epoll 机制监听 EPOLLIN 事件，当有新事件进入消息队列时，并且当前线程处于空闲状态，通过向管道写入数据，来唤醒线程。





# 参考
https://www.cnblogs.com/renhui/p/12875396.html

https://juejin.cn/post/6844904079743795208