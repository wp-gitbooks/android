
# 线索
1. 初始化流程
2. 检测流程

![image-20210416100120875](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100120.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303181651029.png)

## 原理

https://square.github.io/leakcanary/fundamentals-how-leakcanary-works/

1. Detecting retained objects. 发现保留对象
2. Dumping the heap.    dumping当前堆栈信息
3. Analyzing the heap.     分析堆栈信息
4. Categorizing leaks.      内存泄露情况分类


### 核心原理

![image-20210416100120875](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100120.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303181651029.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303181656480.png)


### Reference和ReferenceQueue
[[内存引用-Reference和ReferenceQueue]]

![image-20210416100507414](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100507.png)



![image-20210416100524676](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100524.png)


### 详细的发现保留对象

Set-----> retainedKeys

KeyedReference ------>Key

​										referenceQuenceName

当调用watch函数，先生成一个key，加入到Set中，并且创建一个KeyedReference(key,referneceQuence)，当activity被回收的时候，就会加入到referenceQuence，当从referenceQuence队列出队，并且通过key，在Set中查找，能找到证明还没出队，说明这个是有内存泄露可能。

```plantuml
class RefWatcher {
	Set<String> retainedKeys;
	ReferenceQueue<Object> queue;
}
```


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100548.png)


### gc方式

![image-20210416100829216](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100829.png)

```java
public interface GcTrigger {
    // 默认实现
    GcTrigger DEFAULT = new GcTrigger() {
    @Override 
    public void runGc() {
        // Code taken from AOSP FinalizationTest:
        // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
        // java/lang/ref/FinalizationTester.java
        // System.gc() does not garbage collect every time. Runtime.gc() is
        // more likely to perfom a gc.
        Runtime.getRuntime().gc();//调用Runtime.gc()来执行GC操作
        enqueueReferences();//等待100ms中，等待弱引用对象进入引用队列中
        System.runFinalization();//执行对象的finalize()方法
    }

    private void enqueueReferences() {
        // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
        // references to the appropriate queues.
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            throw new AssertionError();
        }
    }
};

void runGc();
}
```

GcTrigger是一个接口类，里面定义而来runGc()方法，GcTrigger接口里面还定义了一个默认实现DEFAULT。通过GcTrigger的runGc()方法可以主动触发GC操作，这样可以被回收的弱引用对象将会被放入引用队列中，这样后续就可以检查引用队列来判断是否回收成功了。

在GcTrigger的默认实现中，是通过**Runtime.gc()方法**将执行垃圾回收的，因为**System.gc()不能保证每次都执行垃圾收集**。另外，为了确保弱引用对象被放入引用队列中，需要每次垃圾收集后，**等待100ms，让弱引用有足够时间放入到引用队列中**。最后在通过System.runFinalization()方法执行引用对象的finalize()方法

### IdleHandler
```java
  private void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }
```

