---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 参考

http://gityuan.com/2015/12/26/handler-message-framework/



# 2 线索
Handler、Looper、Message、MessageQueue之间的关系？

线程与Handler、Looper之间的关系



what：

​    1、Handler、Looper、Message、MessageQueue的**数据结构**是怎么样的？

​    2、Handler、Looper、Message、MessageQueue的**功能（主要函数）**？





![image-20210524185246183](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210524185246.png)





![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210530215206)

# 3 概述

## 3.1 是什么（用途）

解决**线程间通信**，能够通过Handler把消息从一个线程发送到另外一个线程

[[Handler面试题#1、Handler有哪些作用]]

### 3.1.1 跨线程原理

![image-20210524184322741](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210524184322.png)

[[Handler面试题#13、Handler是如何进行线程切换的呢？]]

### 3.1.2 其他跨线程通信技术

https://redspider.gitbook.io/concurrent/di-yi-pian-ji-chu-pian/5#51-suo-yu-tong-bu

https://zhuanlan.zhihu.com/p/108180480

https://www.cnblogs.com/hapjin/p/5492619.html


### 3.1.3 为什么不建议子线程更新UI？


### 3.1.4 子线程访问UI的崩溃原因和解决方案？

崩溃发生在ViewRootImpl类的`checkThread`方法中：

```java
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }  

```

其实就是判断了当前线程 是否是 `ViewRootImpl`创建时候的线程，如果不是，就会崩溃。

而ViewRootImpl创建的时机就是界面被绘制的时候，也就是onResume之后，所以如果在子线程进行UI更新，就会发现当前线程（子线程）和View创建的线程（主线程）不是同一个线程，发生崩溃。

解决办法有三种：

- 在新建视图的线程进行这个视图的UI更新，主线程创建View，主线程更新View。
- 在`ViewRootImpl`创建之前进行子线程的UI更新，比如onCreate方法中进行子线程更新UI。
- 子线程切换到主线程进行UI更新，比如`Handler、view.post`方法。

## 3.2 组成

- **Message**：消息分为硬件产生的消息(如按钮、触摸)和软件生成的消息；
- **MessageQueue**：消息队列的主要功能向消息池投递消息(<font color='red'>`MessageQueue.enqueueMessage`</font>)和取走消息池的消息(<font color='red'>MessageQueue.next</font>); 
- **Handler**：消息辅助类，主要功能向消息池发送各种消息事件(<font color=red>`Handler.sendMessage`</font>)和处理相应消息事件(<font color='red'>`Handler.handleMessage`</font>)； 
- **Looper**：不断循环执行(<font color=red>`Looper.loop`</font>)，按分发机制将消息分发给目标处理者。



### 3.2.1 关系
Handler、Looper、MessageQueue、线程是一一对应关系吗？

- 一个线程只会有一个`Looper`对象，所以线程和Looper是一一对应的。
- `MessageQueue`对象是在new Looper的时候创建的，所以Looper和MessageQueue是一一对应的。
- `Handler`的作用只是将消息加到MessageQueue中，并后续取出消息后，根据消息的target字段分发给当初的那个handler，所以Handler对于Looper是可以多对一的，也就是多个Hanlder对象都可以用同一个线程、同一个Looper、同一个MessageQueue。

总结：Looper、MessageQueue、线程是一一对应关系，而他们与Handler是可以一对多的。


一一对应：线程---->Looper---->MessageQueue
一对多：多个Hanlder对象都可以用同一个线程、同一个Looper、同一个MessageQueue


### 3.2.2 架构图
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303031432637.png)
- **Looper**有一个MessageQueue消息队列；
- **MessageQueue**有一组待处理的Message；
- **Message**中有一个用于处理消息的Handler；
- **Handler**中有Looper和MessageQueue

## 3.3 消息机制

个人理解：
```
类比：一个传送带(Looper)，通过Handler从传送带的上面放东西，结果传送带传递，从传送带的下面把东西释放出来给Handler。
```

### 3.3.1 流程图
Looper  同   Thread关联

![handler_java](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210310203335.jpg)



![image-20210524184122960](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210524184123.png)



注意：Handler发送的sendMessage当时所在线程同Handler的handleMessage()的线程可能不同，handlerMessage()接收的线程是同Looper所在线程相同的，参考下面的简化流程图

图解：

- Handler通过sendMessage()发送Message到MessageQueue队列；
- Looper通过loop()，不断提取出达到触发条件的Message，并将Message交给target来处理；
- 经过dispatchMessage()后，交回给Handler的handleMessage()来进行相应地处理。
- 将Message加入MessageQueue时，处往管道写入字符，可以会唤醒loop线程；如果MessageQueue中没有Message，并处于Idle状态，则会执行IdelHandler接口中的方法，往往用于做一些清理性地工作


### 3.3.2 简化流程图

**Binder/Socket用于进程间通信，而Handler消息机制用于同进程的线程间通信**，Handler消息机制是由一组MessageQueue、Message、Looper、Handler共同组成的，为了方便且称之为Handler消息机制。


![handler_communication](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210310142802.jpg)


### 3.3.3 生产者-消费者模型

#### 3.3.3.1 简单的生产者-消费者模型

一个简单的生产者-消费者模型如下图所示：

![Simple Producer-Consumer](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423102736.png)

在该模型中，有 4 个主要的角色：

- **消息（Mesage）**：消息当中一般存放诸如消息类型、附加数据等信息，由作为发送方的生产者写入，并由作为接收者的消费者读取。
- **消息队列（Message Queue）**：缓存由生产者插入的消息，并等待消费者摘取。所以消息队列是一个共享队列，在具体的实现中，需要进行判空、判满、并发访问操作。
- **生产者（Producer）**：不定期往消息队列插入一个消息。
- **消费者（Consumer）**：从消息队列中取出一个消息，并执行后续操作。


#### 3.3.3.2 Handler 中的生产者-消费者模型

Handler 继承了生产者-消费者模型的思想，如下图所示：

![Handler Producer-Consumer](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423102755.png)

其要点如下：

- 有两个线程——消费者线程和生产者线程，Handler 以及消息队列隶属于消费者线程。如果从内存模型上来判断，这样表述其实并不准确，我们仅从抽象层次来思考，注意，共享的堆并不隶属于某一个线程。
- 生产者往消息队列中插入消息，是通过持有一个 Handler 引用来实现的。
- 生产者通过持有 Handler 引用，可以访问共享的消息队列，往消息队列中写入消息。
- 消费者从共享队列中取消息，是通过 Handler 不停地访问消息队列来实现的。也就是我们在很多博客中看到的 Looper 中的死循环检查。
- 所以，具体实现的时候，通常都是 Handler 提供一个接口给消费者，这样 Handler 便持有了一个消费者的引用，当 Handler 取到消息时，便通过该接口将消息传递给消费者。这通常又引入了 android 当中一个臭名昭著的 Context 泄露问题


### 3.3.4 实例

```java
class LooperThread extends Thread {
    public Handler mHandler;

    public void run() {
        Looper.prepare();  

        mHandler = new Handler() { 
            public void handleMessage(Message msg) {
                //TODO 定义消息处理逻辑. 
            }
        };

        Looper.loop(); 
    }
}
```

### 3.3.5 对比多线程 [[线程池-原理]]
从实现原理上进行对比

# 4 Handler(Java层)

## 4.1 什么是Handler

**Handler机制是Android中基于单线消息队列模式的一套线程消息机制。**

### 4.1.1 基本用法

```
//在主线程创建Handler实例
private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(@NonNull Message msg) {
            //处理接收的消息
        }
    };

//在适当的时机使用Handler实例发送消息
mHandler.sendMessage(message);
mHandler.post(runnable);//Runnable会被封装进一个Message，所以它本质上还是一个Message
```


## 4.2 创建Handler

```java
Handler()
 
Handler(Callback callback)
 
Handler(Looper looper)
 
Handler(Looper looper, Callback callback)
 
Handler(boolean async)
 
Handler(Callback callback, boolean async)
 
Handler(Looper looper, Callback callback, boolean async)
```


### 4.2.1 无参构造

> 这种就需要在创建Handler前，预先调用Looper.prepare来创建当前线程的默认Looper，否则会报错。

```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
    //匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    //必须先执行Looper.prepare()，才能获取Looper对象，否则为null.
    mLooper = Looper.myLooper();  //从当前线程的TLS中获取Looper对象
    if (mLooper == null) {
        throw new RuntimeException("");
    }
    mQueue = mLooper.mQueue; //消息队列，来自Looper对象
    mCallback = callback;  //回调方法
    mAsynchronous = async; //设置消息是否为异步处理方式
}
```

对于Handler的无参构造方法，默认采用当前线程TLS中的Looper对象，并且callback回调方法为null，且消息为同步处理方式。只要执行的Looper.prepare()方法，那么便可以获取有效的Looper对象。


### 4.2.2 有参构造

> 这种就是Handler和指定的Looper进行绑定，也就是说Handler其实是可以跟任意线程进行绑定的，不局限于在创建Handler所在的线程里。

```java
public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

```


## 4.3 消息处理
![image-20210629182841978](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210629182842.png)

### 4.3.1 消息发送(sendMessage、postRunnable) 
普通消息、空消息、延迟消息、指定时间消息、发送到队列最前面消息
1、空消息用途？
2、延迟消息实现？

![image-20210524184045429](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210629182441.png)

![java_sendmessage](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423105249.png)

```plantuml
Handler -> Handler:sendMesage
Handler -> Handler:sendMessageDelayed
Handler -> Handler:sendMessageAtTime
Handler -> Handler:enqueueMessage
Handler -> MessageQueue:enqueueMessage
```


#### 4.3.1.1 message

##### 4.3.1.1.1 sendEmptyMessage

```java
public final boolean sendEmptyMessage(int what) {
    return sendEmptyMessageDelayed(what, 0);
}
```

##### 4.3.1.1.2 sendEmptyMessageDelayed

```java
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}
```

##### 4.3.1.1.3 sendMessageDelayed

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

##### 4.3.1.1.4 sendMessageAtTime

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

##### 4.3.1.1.5 sendMessageAtFrontOfQueue

```java
public final boolean sendMessageAtFrontOfQueue(Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, 0);
}
```

该方法通过设置消息的触发时间为0，从而使Message加入到消息队列的队头。


#### 4.3.1.2 Runnable

##### 4.3.1.2.1 post

```java
public final boolean post(Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

##### 4.3.1.2.2 postDelayed

```java
public final boolean postDelayed(Runnable r, long delayMillis)
{
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}
```


##### 4.3.1.2.3 postAtTime

```java
public final boolean postAtTime(Runnable r, long uptimeMillis)
{
    return sendMessageAtTime(getPostMessage(r), uptimeMillis);
}
```

##### 4.3.1.2.4 postAtFrontOfQueue

```java
public final boolean postAtFrontOfQueue(Runnable r) {
    return sendMessageAtFrontOfQueue(getPostMessage(r));
}
```


#### 4.3.1.3 post(Runnable) 与 sendMessage 有什么区别
[[Handler面试题#Handler 的 post Runnable 与 sendMessage 有什么区别]]

#### 4.3.1.4 enqueueMessage

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis); 【见4.3】
}
```


### 4.3.2 消息移除

```java
public final void removeMessages(int what) {
    mQueue.removeMessages(this, what, null);
}
```


```java
public final void removeMessages(int what, Object object) {
    mQueue.removeMessages(this, what, object);
}
```


```java
public final void removeCallbacks(Runnable r)
{
    mQueue.removeMessages(this, r, null);
}
```


```java
public final void removeCallbacks(Runnable r, Object token)
{
    mQueue.removeMessages(this, r, token);
}
```


```java
public final void removeCallbacksAndMessages(Object token) {
    mQueue.removeCallbacksAndMessages(this, token);
}
```

### 4.3.3 消息查询
```java
public final boolean hasMessages(int what) {
    return mQueue.hasMessages(this, what, null);
}
```

```java
public final boolean hasMessages(int what, Object object) {
    return mQueue.hasMessages(this, what, object);
}
```

```java
public final boolean hasCallbacks(Runnable r) {
    return mQueue.hasMessages(this, r, null);
}
```

```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}

private static Message getPostMessage(Runnable r, Object token) {
    Message m = Message.obtain();
    m.obj = token;
    m.callback = r;
    return m;
```


## 4.4 消息分发机制

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423153046)

在Looper.loop()中，当发现有消息时，调用消息的目标handler，执行dispatchMessage()方法来分发消息。

```java
Message msg = queue.next(); 
msg.target.dispatchMessage(msg);
```



```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        //当Message存在回调方法，回调msg.callback.run()方法；
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            //当Handler存在Callback成员变量时，回调方法handleMessage()；
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //Handler自身的回调方法handleMessage()
        handleMessage(msg);
    }
}
```

### 4.4.1 分发消息流程

1. 当`Message`的回调方法不为空时，则回调方法`msg.callback.run()`，其中callBack数据类型为Runnable,否则进入步骤2；
2. 当`Handler`的`mCallback`成员变量不为空时，则回调方法`mCallback.handleMessage(msg)`,否则进入步骤3；
3. 调用`Handler`自身的回调方法`handleMessage()`，该方法默认为空，Handler子类通过覆写该方法来完成具体的逻辑。

对于很多情况下，消息分发后的处理方法是第3种情况，即Handler.handleMessage()，一般地往往通过覆写该方法从而实现自己的业务逻辑。

### 4.4.2 消息分发的优先级

1. Message的回调方法：`message.callback.run()`，优先级最高；   

   ```java
   （通过Handler.post(Runnable r)）
   ```

   

2. Handler的回调方法：`Handler.mCallback.handleMessage(msg)`，优先级仅次于1； 

   ```
    (通过构造函数public Handler(Callback callback) )
   ```


3. Handler的默认方法：`Handler.handleMessage(msg)`，优先级最低   

   ```java
   Handler handler = new Handler(){
               @Override
               public void handleMessage(Message msg) {
                   super.handleMessage(msg);
   
               }
           };
   ```

   
## 4.5 Handler内存泄露原因? 如何解决？
[[Handler内存泄露原因? 如何解决？]]

## 4.6 Handler怎么进行线程通信，原理是什么？
[[#3 1 1 跨线程原理]]

## 4.7 ActivityThread中做了哪些关于Handler的工作？（为什么主线程不需要单独创建Looper）
主要做了两件事：
- 1、在main方法中，创建了主线程的`Looper`和`MessageQueue`，并且调用loop方法开启了主线程的消息循环。

```java
public static void main(String[] args) {

        Looper.prepareMainLooper();

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

```

- 2、创建了一个Handler来进行四大组件的启动停止等事件处理

```java
final H mH = new H();

class H extends Handler {
        public static final int BIND_APPLICATION        = 110;
        public static final int EXIT_APPLICATION        = 111;
        public static final int RECEIVER                = 113;
        public static final int CREATE_SERVICE          = 114;
        public static final int STOP_SERVICE            = 116;
        public static final int BIND_SERVICE            = 121;

```

## 4.8 Handler发送消息的时候，时间为啥要取SystemClock.uptimeMillis() + delayMillis，可以把SystemClock.uptimeMillis() 换成System.currentTimeMillis()吗？
答：不可以

**SystemClock.uptimeMillis()** 这个方法获取的时间，是自系统开机到现在的一个毫秒数，这个时间是个相对的

**System.currentTimeMillis()** 这个方法获取的是自**1970-01-01 00:00:00** 到现在的一个毫秒数，这是一个和系统强关联的时间，而且这个值可以做修改

1、使用System.currentTimeMillis()可能会导致延迟消息失效

2、最终这个时间会被设置到Message的when属性，而Message的when属性只是需要一个时间差来表示消息的先后顺序，使用一个相对时间就行了，没必要使用一个绝对时间


## 4.9 Handler如何处理发送延迟消息？[[#6 1 2 2 阻塞IO的事件通信机制]]
由底层epoll机制延迟实现

nativePollOnce    //block阻塞

nativeWake

![image-20210708153954547](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210708153954.png)

## 4.10 Handler价值：卡顿监控 [[BlockCanary]]  
https://juejin.cn/post/7179959600001581116
https://zhuanlan.zhihu.com/p/591840032



## 4.11 Handler能做定时器嘛？

不可以，因为每个消息执行时间不定，等到自己发送的消息执行的时间就不确定。（handler机制导致不准确，不是nativePollOnce导致不准确）

## 4.12 Handler.post和View.post


## 4.13 利用Handler机制设计一个不崩溃的App？
主线程崩溃，其实都是发生在消息的处理内，包括生命周期、界面绘制。
所以如果我们能控制这个过程，并且在发生崩溃后重新开启消息循环，那么主线程就能继续运行。

```java
Handler(Looper.getMainLooper()).post {
        while (true) {
            //主线程异常拦截
            try {
                Looper.loop()
            } catch (e: Throwable) {
            }
        }
    }

```


# 5 Looper [[Android中为什么主线程不会因为Looper.loop()里的死循环卡死？]]



## 5.1 基础知识

```plantuml
Looper -> Looper:loop
Looper -> MessageQueue:next
Looper -> Message:target
Message -> Handler:dispatchMessage
```

### 5.1.1 ThreadLocal
[[8-ThreadLocal]]


## 5.2 初始化

### 5.2.1 prepare()和prepareMainLooper---Looper创建

对于无参的情况，默认调用`prepare(true)`，表示的是这个Looper允许退出，而对于false的情况则表示当前Looper不允许退出。Handler机制用到的跟Thread相关的，而根本原因是Handler必须和对应的Looper绑定，而Looper的创建和保存是跟Thread一一对应的，也就是说每个线程都可以**创建唯一一个且互不相关的Looper**，这是通过ThreadLocal来实现的，也就是说是用ThreadLocal对象来存储Looper对象的，从而达到线程隔离的目的

```Java
private static void prepare(boolean quitAllowed) {
    //每个线程只允许执行一次该方法，第二次执行时线程的TLS已有数据，则会抛出异常。
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //创建Looper对象，并保存到当前线程的TLS区域
    sThreadLocal.set(new Looper(quitAllowed));
}
```

这里的`sThreadLocal`是ThreadLocal类型，下面，先说说ThreadLocal。

**ThreadLocal**： 线程本地存储区（Thread Local Storage，简称为TLS），每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。TLS常用的操作方法：

- `ThreadLocal.set(T value)`：将value存储到当前线程的TLS区域，源码如下：

```Java
public void set(T value) {
    Thread currentThread = Thread.currentThread(); //获取当前线程
    Values values = values(currentThread); //查找当前线程的本地储存区
    if (values == null) {
        //当线程本地存储区，尚未存储该线程相关信息时，则创建Values对象
        values = initializeValues(currentThread);
    }
    //保存数据value到当前线程this
    values.put(this, value);
}
```

- `ThreadLocal.get()`：获取当前线程TLS区域的数据，源码如下：

```Java
public T get() {
    Thread currentThread = Thread.currentThread(); //获取当前线程
    Values values = values(currentThread); //查找当前线程的本地储存区
    if (values != null) {
        Object[] table = values.table;
        int index = hash & values.mask;
        if (this.reference == table[index]) {
            return (T) table[index + 1]; //返回当前线程储存区中的数据
        }
    } else {
        //创建Values对象
        values = initializeValues(currentThread);
    }
    return (T) values.getAfterMiss(this); //从目标线程存储区没有查询是则返回null
}
```

ThreadLocal的get()和set()方法操作的类型都是泛型，接着回到前面提到的`sThreadLocal`变量，其定义如下：

```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>()
```

可见`sThreadLocal`的get()和set()操作的类型都是`Looper`类型。

**Looper.prepare()**

Looper.prepare()在每个线程只允许执行一次，该方法会创建Looper对象，Looper的构造方法中会创建一个MessageQueue对象，再将Looper对象保存到当前线程TLS。

对于Looper类型的构造方法如下：

```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);  //创建MessageQueue对象. 【见4.1】
    mThread = Thread.currentThread();  //记录当前线程.
}
```

另外，与prepare()相近功能的，还有一个`prepareMainLooper()`方法，该方法主要在ActivityThread类中使用。

```
public static void prepareMainLooper() {
    prepare(false); //设置不允许退出的Looper
    synchronized (Looper.class) {
        //将当前的Looper保存为主Looper，每个线程只允许执行一次。
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```


### 5.2.2 Thread和Looper关系
Looper是依附在Thread之上的。一个Thread 对一个Looper

```java
public final class Looper {
  static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
   private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
  }
}
```

调用Looper.prepare()创建Looper后是保存在sThreadLocal中。


### 5.2.3 Looper如何在子线程中创建？
```java
    Handler mHandler;
        new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();//Looper初始化
                //Handler初始化 需要注意, Handler初始化传入Looper对象是子线程中缓存的Looper对象
                mHandler = new Handler(Looper.myLooper());
                Looper.loop();//死循环
                //注意: Looper.loop()之后的位置代码在Looper退出之前不会执行,(并非永远不执行)
            }
        }).start();
```


## 5.3 获取

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```


```java
public static Looper getMainLooper() {
    synchronized (Looper.class) {
        return sMainLooper;
    }
}
```


## 5.4 线程

```java
public boolean isCurrentThread() {
    return Thread.currentThread() == mThread;
}
```

```java
public @NonNull Thread getThread() {
    return mThread;
}
```


## 5.5 loop()---Looper开启消息循环

```java
public static void loop() {
    final Looper me = myLooper();  //获取TLS存储的Looper对象 
    final MessageQueue queue = me.mQueue;  //获取Looper对象中的消息队列

    Binder.clearCallingIdentity();
    //确保在权限检查时基于本地进程，而不是调用进程。
    final long ident = Binder.clearCallingIdentity();

    for (;;) { //进入loop的主循环方法
        Message msg = queue.next(); //可能会阻塞 
        if (msg == null) { //没有消息，则退出循环
            return;
        }

        //默认为null，可通过setMessageLogging()方法来指定输出，用于debug功能
        Printer logging = me.mLogging;  
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        msg.target.dispatchMessage(msg); //用于分发Message 
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        //恢复调用者信息
        final long newIdent = Binder.clearCallingIdentity();
        msg.recycleUnchecked();  //将Message放入消息池 
    }
}
```

loop()进入循环模式，不断重复下面的操作，直到没有消息时退出循环

- `读取`MessageQueue的下一条Message；
- 把Message`分发`给相应的target；
- 再把分发后的Message`回收`到消息池，以便重复利用。

这是这个消息处理的核心部分。另外，上面代码中可以看到有logging方法，这是用于debug的，默认情况下`logging == null`，通过设置setMessageLogging()用来开启debug工作。

## 5.6 quit() ---Looper退出

消息退出的方式：

- **当safe =true时，只移除尚未触发的所有消息**，对于正在触发的消息并不移除；
- 当safe =flase时，移除所有的消息

```
public void quit() {
    mQueue.quit(false); //消息移除
}

public void quitSafely() {
    mQueue.quit(true); //安全地消息移除
}
```

Looper.quit()方法的实现最终调用的是MessageQueue.quit()方法

**MessageQueue.quit()**

```
void quit(boolean safe) {
        // 当mQuitAllowed为false，表示不运行退出，强行调用quit()会抛出异常
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }
        synchronized (this) {
            if (mQuitting) { //防止多次执行退出操作
                return;
            }
            mQuitting = true;
            if (safe) {
                removeAllFutureMessagesLocked(); //移除尚未触发的所有消息
            } else {
                removeAllMessagesLocked(); //移除所有的消息
            }
            //mQuitting=false，那么认定为 mPtr != 0
            nativeWake(mPtr);
        }
    }
```

## 5.7 卡顿监控

```java
public void setMessageLogging(@Nullable Printer printer) {  
    mLogging = printer;  
}
```

# 6 MessageQueue

**MessageQueue中最重要的就是两个方法**：
1.enqueueMessage向队列中`插入`消息
2.next 从队列中`取出`消息


MessageQueue是消息机制的Java层和C++层的连接纽带，大部分核心方法都交给native层来处理，其中MessageQueue类中涉及的native方法如下：

```
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

关于这些native方法的介绍，见[Android消息机制2-Handler(native篇)](http://gityuan.com/2015/12/27/handler-message-native/)。


## 6.1 消息处理

### 6.1.1 消息加入:enqueMesage [[1-大话数据结构-线性表#头插法]]

添加一条消息到消息队列:通过when时间
1、队头插入
2、遍历队列插入

```Java
boolean enqueueMessage(Message msg, long when) {
    // 每一个普通Message必须有一个target
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {  //正在退出时，回收msg，加入到消息池
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //p为null(代表MessageQueue没有消息） 或者msg的触发时间是队列中最早的， 则进入该该分支
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; //当阻塞时需要唤醒
        } else {
            //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，除非
            //消息队头存在barrier，并且同时Message是队列中最早的异步消息。
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p;
            prev.next = msg;
        }
        //消息没有退出，我们认为此时mPtr != 0
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

`MessageQueue`是按照**Message触发时间的先后顺序**排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。

#### 6.1.1.1 详解

**1、将新来的Message消息 插入到链表中合适的位置。**

满足以下条件，新message会被插入到链表头部

- a、当前队列没有待处理的消息
- b、新消息msg.when = 0（这种情况一般不存在)
- c、新消息的触发时间 早于mMessages表头消息的触发时间
  其他情况,新message会被插入到链表中部合适位置。

**2、满足条件时，唤醒队列**

```undefined
 if (needWake) {
    nativeWake(mPtr);
 }
```

什么时候需要唤醒队列:

- 新消息**插入链表头部**时，需要立即唤醒队列
- 新消息插入链表中部时，一般不需要立即唤醒;但是当链表表头是一个**消息屏障，新先消息是一个异步消息**时，才需要唤醒队列。


#### 6.1.1.2 延时消息实现
通过when时间来实现

MessageQueue中的消息 是一个单向链表的形式保存的,mMessages 变量 指向链表的表头。Message链表中的元素是以**触发时间(when)为基准,从小道大排列的,when小的排在链表前面**,优先被处理；when大的排在链表的后面,延后处理。


##### 6.1.1.2.1 Handler发送消息的delay设置是否可靠？
答案是：不可靠。

原因：当Handler所属的线程（UI线程）要处理的内容非常多，当Looper出现事件积压的时候会使得delay不可靠。如ANR的出现就是一个最极端的代表例子。

为了理解为何在事件挤压的时候，handler会出现delay的不可靠，这里我们就加入一些核心逻辑的分析，来帮助我们进行此问题的探究。


### 6.1.2 消息循环:next()

提取下一条message

```java
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) { //当消息循环已经退出，则直接返回
        return null;
    }
    int pendingIdleHandlerCount = -1; // 循环迭代的首次为-1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        //阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //当消息的Handler为空时，则查询异步消息
            if (msg != null && msg.target == null) {
                //当查询到异步消息，则立刻退出循环
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取一条消息，并返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    //设置消息的使用状态，即flags |= FLAG_IN_USE
                    msg.markInUse();
                    return msg;   //成功地获取MessageQueue中的下一条即将要执行的消息
                }
            } else {
                //没有消息
                nextPollTimeoutMillis = -1;
            }
            //消息正在退出，返回null
            if (mQuitting) {
                dispose();
                return null;
            }
            //当消息队列为空，或者是消息队列的第一个消息时
            if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                //没有idle handlers 需要运行，则循环并等待。
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        //只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; //去掉handler的引用
            boolean keep = false;
            try {
                keep = idler.queueIdle();  //idle时执行的方法
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        //重置idle handler个数为0，以保证不会再次重复运行
        pendingIdleHandlerCount = 0;
        //当调用一个空闲handler时，一个新message能够被分发，因此无需等待可以直接查询pending message.
        nextPollTimeoutMillis = 0;
    }
}
```

`nativePollOnce`是阻塞操作，其中`nextPollTimeoutMillis`代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。

当处于空闲时，往往会执行`IdleHandler`中的方法。当nativePollOnce()返回后，next()从`mMessages`中提取一个消息。

`nativePollOnce()`在native做了大量的工作，想进一步了解可查看 [Android消息机制2-Handler(native篇)](http://gityuan.com/2015/12/27/handler-message-native/#nativepollonce)。

#### 6.1.2.1 详解

##### 6.1.2.1.1 nativePollOnce()

通过Native层的epoll来阻塞住当前线程

```java
 nativePollOnce(ptr, nextPollTimeoutMillis);
 private native void nativePollOnce(long ptr, int timeoutMillis)
```

nativePollOnce根据传入的参数nextPollTimeoutMillis 会有不同的阻塞行为

- 如果nextPollTimeoutMillis=-1，一直阻塞不会超时。
- 如果nextPollTimeoutMillis=0，不会阻塞，立即返回。
- 如果nextPollTimeoutMillis>0，最长阻塞nextPollTimeoutMillis毫秒(超时)，如果期间有程序唤醒会立即返回。

##### 6.1.2.1.2 处理同步屏障消息
[[2-Handler同步屏障]]
如果链表表头是一个同步屏障消息,则跳过众多同步消息,找到链表中第一个异步消息,进行分发处理

Message 分为同步消息和异步消息。

```java
class Message{
    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
    }
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
}
```

通常我们使用Handler发消息都是同步消息，发出去之后就会在消息队列里面排队处理。我们都知道，Android系统16ms会刷新一次屏幕，如果主线程的消息过多，在16ms之内没有执行完，必然会造成卡顿或者掉帧。那怎么才能不排队，没有延时的处理呢？这个时候就需要异步消息。在处理异步消息的时候，我们就需要同步屏障，让异步消息不用排队等候处理。

可以理解为 **<font color=red>同步屏障是一堵墙，把同步消息队列拦住，先处理异步消息</font>**，等异步消息处理完了，这堵墙就会取消，然后继续处理同步消息。

MessageQueue里面有postSyncBarrier()可以发送同步屏障消息

```csharp
public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

值得注意的是，**<font color=red>同步屏障消息没有target，普通的消息的必须有target的</font>**。

再回到MessageQueue.next() 方法

```java
//(2)如果链表表头是一个同步屏障消息,则跳过众多同步消息,找到链表中第一个异步消息,进行分发处理
        if (msg != null && msg.target == null) {
            // Stalled by a barrier.  Find the next asynchronous message in the queue.
            do {
                prevMsg = msg;
                msg = msg.next;
            } while (msg != null && !msg.isAsynchronous());
        }
```

我们看到,如果链表表头是一个同步屏障消息,就会遍历链表,返回第一个待处理异步消息。这样就跳过了前面众多的同步消息。

##### 6.1.2.1.3 取到的Message返回还是阻塞?

```csharp
//(3) 返回取到的Message 
        if (msg != null) {
            //msg尚未到达触发时间,则计算新的阻塞超时时间nextPollTimeoutMillis，下次循环触发队列阻塞
            if (now < msg.when) {
                // Next message is not ready.  Set a timeout to wake up when it is ready.
                nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
            } else {
                //从链表中移除该消息后，直接返回
                // Got a message.
                mBlocked = false;
                if (prevMsg != null) {
                    prevMsg.next = msg.next;
                } else {
                    mMessages = msg.next;
                }
                msg.next = null;
                msg.markInUse();
                return msg;
            }
        } else { //没有找到异步消息,设置nextPollTimeoutMillis=-1,队列阻塞
            // No more messages.
            nextPollTimeoutMillis = -1;
        }
```

回到next()中的代码块,取到待处理的message后做了如下处理

- 如果msg == null,说明链表中没有消息,则nextPollTimeoutMillis = -1，下次循环 会无限阻塞。
- msg !=null,并且msg.when 符合触发条件,则直接返回
- msg !=null,但是msg.when 尚未到达预期的触发时间点,则重新计算nextPollTimeoutMillis，下次循环进行固定时长的阻塞。

##### 6.1.2.1.4 mQuitting 的处理

如果调用了MessageQueue.quit() ,mQuitting = true,队列中所有的消息都会处理后,会调用dispose 释放MessageQueue


```kotlin
if (mQuitting) {
    dispose();
    return null;
}
```

##### 6.1.2.1.5 对已注册的HandlerIdle回调的处理

MessageQueue可以注册HandlerIdle监听,此处对注册的HandlerIdle做回调处理。

```java
 //(5) 对已注册的HandlerIdle回调的处理
        
        if (pendingIdleHandlerCount < 0
                && (mMessages == null || now < mMessages.when)) {
            pendingIdleHandlerCount = mIdleHandlers.size();
        }
        if (pendingIdleHandlerCount <= 0) {
            // No idle handlers to run.  Loop and wait some more.
            mBlocked = true;
            continue;
        }

        if (mPendingIdleHandlers == null) {
            mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
        }
        mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
                
                
                
                
         for (int i = 0; i < pendingIdleHandlerCount; i++) {
        final IdleHandler idler = mPendingIdleHandlers[i];
        mPendingIdleHandlers[i] = null; // release the reference to the handler

        boolean keep = false;
        try {
            keep = idler.queueIdle();
        } catch (Throwable t) {
            Log.wtf(TAG, "IdleHandler threw exception", t);
        }

        if (!keep) {
            synchronized (this) {
                mIdleHandlers.remove(idler);
            }
        }
    }
```


#### 6.1.2.2 阻塞IO的事件通信机制

唤醒：enqueueMessage：nativeWake

阻塞：next：nativePollOnce


从select、poll、epool机制

![Blocked IO](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423104036.png)



##### 6.1.2.2.1 主线程进入loop循环了为什么没有ANR？ [[Android中为什么主线程不会因为Looper.loop()里的死循环卡死？]]

[[Handler机制与管道Pipe机制]]


https://www.zhihu.com/question/34652589/answer/90344494


**(1) Android中为什么主线程不会因为Looper.loop()里的死循环卡死？** 

这里涉及线程，先说说说进程/线程，**进程：**每个app运行时前首先创建一个进程，该进程是由Zygote fork出来的，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，让App程序都是运行在Android Runtime。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。

**线程：**线程对应用来说非常常见，比如每次new Thread().start都会创建一个新的线程。该线程与App所在进程之间资源共享，从Linux角度来说进程与线程除了是否共享资源外，并没有本质的区别，都是一个task_struct结构体**，在CPU看来进程或线程无非就是一段可执行的代码，CPU采用CFS调度算法，保证每个task都尽可能公平的享有CPU时间片**。

有了这么准备，再说说死循环问题：

对于线程既然是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？**简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，**例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。

真正会卡死主线程的操作是在回调方法onCreate/onStart/onResume等操作时间过长，会导致掉帧，甚至发生ANR，looper.loop本身不会导致应用卡死。

**主线程的死循环一直运行是不是特别消耗CPU资源呢？** 其实不然，这里就涉及到**Linux pipe/e**poll机制**，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，详情见[Android消息机制1-Handler(Java层)](https://link.zhihu.com/?target=http%3A//www.yuanhh.com/2015/12/26/handler-message-framework/%23next)，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 **所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源**





#### 6.1.2.3 其他：线程安全

我们可以使用多个Handler往消息队列中添加数据，那么可能存在发消息的Handler存在不同的线程，那么Handler是如何保证MessageQueue并发访问安全的呢？

答：循环加锁，配合阻塞唤醒机制

我们可以发现MessageQueue其实是“生产者-消费者”模型，Handler不断地放入消息，Looper不断地取出，这就涉及到死锁问题。如果Looper拿到锁，但是队列中没有消息，就会一直等待，而Handler需要把消息放进去，锁却被Looper拿着无法入队，这就造成了死锁。Handler机制的解决方法是**循环加锁**。在MessageQueue的next方法中：


```java
Message next() {
   ...
    for (;;) {
  ...
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            ...
        }
    }
}
```

我们可以看到他的等待是在锁外的，当队列中没有消息的时候，他会先释放锁，再进行等待，直到被唤醒。这样就不会造成死锁问题了。

那在入队的时候会不会因为队列已经满了然后一边在等待消息处理一边拿着锁呢？这一点不同的是MessageQueue的消息没有上限，或者说他的上限就是JVM给程序分配的内存，如果超出内存会抛出异常，但一般情况下是不会的。


### 6.1.3 消息查询

hasMessage


### 6.1.4 消息删除

removeMessage、removeCallbackMessage


## 6.2 消息队列退出

quitAllowed

quit

消息退出的方式：

- **当safe =true时，只移除尚未触发的所有消息**，对于正在触发的消息并不移除；
- 当safe =flase时，移除所有的消息

**MessageQueue.quit()**

```
void quit(boolean safe) {
        // 当mQuitAllowed为false，表示不运行退出，强行调用quit()会抛出异常
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }
        synchronized (this) {
            if (mQuitting) { //防止多次执行退出操作
                return;
            }
            mQuitting = true;
            if (safe) {
                removeAllFutureMessagesLocked(); //移除尚未触发的所有消息
            } else {
                removeAllMessagesLocked(); //移除所有的消息
            }
            //mQuitting=false，那么认定为 mPtr != 0
            nativeWake(mPtr);
        }
    }
```



## 6.3 IdleHandler
[[1-Handler之IdleHandler]]

addIdleHandler

removeIdleHandler



## 6.4 消息屏障Barrier [[2-Handler同步屏障]]

postSyncBarrier

removeSyncBarrier



## 6.5 Handler机制下消息队列MessageQueue的优化
一般情况下，可以考虑从以下几个方面进行优化

-   对消息队列中**重复消息**的过滤，用于控制一些操作的频率，既保证用户体验又保证性能（使用Handler Api）。
-   对消息队列中的互斥消息进行取消，用于类似开关之类的操作（使用Handler Api）。
-   **消息池复用**，减少创建Message实例的开销（Handler机制自带消息池）。
-   IdleHandler的合理使用，利用**空闲**的时机进行一些业务逻辑的处理（视业务而定）。


# 7 Message

## 7.1 消息对象

每个消息用`Message`表示，`Message`主要包含以下内容：

| 数据类型 | 成员变量 | 解释                 |
| -------- | :------- | :------------------- |
| int      | what     | 消息类别             |
| long     | when     | 消息触发时间         |
| int      | arg1     | 参数1                |
| int      | arg2     | 参数2                |
| Object   | obj      | 消息内容             |
| Bundle   | data     | 数据内容             |
| long     | when     | 消息执行时间         |
| int      | flags    | 消息类型：同步、异步 |

其他
| 数据类型     | 成员变量 | 解释         |
| :----------- | :------- | :----------- |
| **Handler**  | target   | 消息响应方   |
| **Runnable** | callback | 回调方法     |

创建消息的过程，就是填充消息的上述内容的一项或多项。

## 7.2 消息池
在代码中，可能经常看到recycle()方法，咋一看，可能是在做虚拟机的gc()相关的工作，其实不然，这是用于把消息加入到消息池的作用。这样的好处是，当消息池不为空时，可以直接从消息池中获取Message对象，而不是直接创建，提高效率。

静态变量`sPool`的数据类型为Message，通过next成员变量，维护一个消息池；静态变量`MAX_POOL_SIZE`代表消息池的可用大小；消息池的默认大小为50。

消息池常用的操作方法是obtain()和recycle()。

### 7.2.1 消息缓存
为了提供效率，提供了一个大小为50的Message缓存队列，减少对象不断创建与销毁的过程。

### 7.2.2 obtain
从消息池中获取消息

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null; //从sPool中取出一个Message对象，并消息链表断开
            m.flags = 0; // 清除in-use flag
            sPoolSize--; //消息池的可用大小进行减1操作
            return m;
        }
    }
    return new Message(); // 当消息池为空时，直接创建Message对象
}
```

obtain()，从消息池取Message，都是把消息池表头的Message取走，再把表头指向next;

### 7.2.3 recycle
把不再使用的消息加入消息池

```java
public void recycle() {
    if (isInUse()) { //判断消息是否正在使用
        if (gCheckRecycle) { //Android 5.0以后的版本默认为true,之前的版本默认为false.
            throw new IllegalStateException("This message cannot be recycled because it is still in use.");
        }
        return;
    }
    recycleUnchecked();
}

//对于不再使用的消息，加入到消息池
void recycleUnchecked() {
    //将消息标示位置为IN_USE，并清空消息所有的参数。
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) { //当消息池没有满时，将Message对象加入消息池
            next = sPool;
            sPool = this;
            sPoolSize++; //消息池的可用大小进行加1操作
        }
    }
}
```

recycle()，将Message加入到消息池的过程，都是把Message加到链表的表头；


## 7.3 其他

### 7.3.1 异步消息

```java
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }

```


# 8 总结

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210524110043.png)



- Handler：事件的发送及处理者，在构造方法中可以设置其 async，默认为 false。若 async 为 true 则该 Handler 发送的 Message 均为异步消息，有同步屏障的情况下会被优先处理。
- Looper：一个用于遍历 MessageQueue 的类，每个线程有一个独有的 Looper，它会在所处的线程开启一个死循环，不断从 MessageQueue 中拿出消息，并将其发送给 target 进行处理
- MessageQueue：用于存储 Message，内部维护了 Message 的链表，每次拿取 Message 时，若该 Message 离真正执行还需要一段时间，会通过 nativePollOnce 进入阻塞状态，避免资源的浪费。若存在消息屏障，则会忽略同步消息优先拿取异步消息，从而实现异步消息的优先消费。

