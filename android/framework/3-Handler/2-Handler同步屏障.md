# 参考

https://www.jianshu.com/p/4a58568661e8



https://segmentfault.com/a/1190000038350183

# 概述

## 消息分类

Handler中的Message可以分为两类：**同步消息、异步消息**。消息类型可以通过以下函数得知

```java
//Message.java
public boolean isAsynchronous() {
    return (flags & FLAG_ASYNCHRONOUS) != 0;
}
```

一般情况下这两种消息的处理方式没什么区别，只有在设置了同步屏障时才会出现差异



*<font color=red>屏障消息和普通消息的区别是屏障消息没有tartget</font>*



### 消息构建（如何发送同步、异步消息）--Handler

**消息按照从小到大进行排列，链表最前面最小**



存到MessageQueue里的消息可能有三种：**同步消息，异步消息，屏障消息**

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210330221223.png)



这里Handler有个async参数，通过这个参数表明通过这个**Handler**发送的消息全都是异步消息，因为在把消息压入队列的时候，会把这个标志设置到message里.这个标志是**全局**的，也就是说通过构造Handler函数传入的async参数，就确定了通过这个Handler发送的消息都是异步消息，默认是false，即都是同步消息

```
public Handler(boolean async);
public Handler(Callback callback, boolean async);
public Handler(Looper looper, Callback callback, boolean async);
```


#### 同步消息
我们默认用的都是同步消息，即前面讲Handler里的**构造函数参数**的async参数默认是false，同步消息在MessageQueue里的存和取完全就是按照时间排的，也就是通过**msg.when**来排的。



#### 异步消息

异步消息就是在创建**Handler如果传入的async是true**或者发送来的Message通过**msg.setAsynchronous(true);**后的消息就是异步消息，异步消息的功能要配合下面要讲的屏障消息才有效，否则和同步消息是一样的处理。

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    // 这个mAsynchronous就是在创建Handler的时候传入async参数
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```







## 什么是同步屏障(同步屏障的插入、删除操作)--MessageQueue
从代码层面上来讲，**同步屏障就是一个Message，一个target字段为空的Message**
同步屏障可以通过MessageQueue.postSyncBarrier函数来设置

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210423142533.png)

### postSyncBarrier
```java
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
            // 这里是说如果p指向的消息时间戳比屏障消息小，说明这个消息比屏障消息先进入队列，
            // 那么这个消息不应该受到屏障消息的影响（屏障消息只影响比它后加入消息队列的消息），找到第一个比屏障消息晚进入的消息指针
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        // 上面找到第一个比屏障消息晚进入的消息指针之后，把屏障消息插入到消息队列中，也就是屏障消息指向第一个比它晚进入的消息p，
        // 上一个比它早进入消息队列的prev指向屏障消息，这样就完成了插入。
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
        // 如果prev是null，说明上面没有经过移动，也就是屏障消息就是在消息队列的头部了。
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
 
```

### removeSyncBarrier

```
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        // 前面在插入屏障消息后会生成一个token，这个token就是用来删除该屏障消息用的。
        // 所以这里通过判断target和token来找到该屏障消息，从而进行删除操作
        // 找到屏障消息的指针p
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        // 上面找到屏障消息的指针p后，把前一个消息指向屏障消息的后一个消息，这样就把屏障消息移除了
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();
 
        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```



可以看到，该函数仅仅是创建了一个Message对象并加入到了消息链表中。乍一看好像没什么特别的，但是这里面有一个很大的不同点是该Message没有target。

我们通常都是通过Handler发送消息的，Handler中发送消息的函数有post***、sendEmptyMessage***以及sendMessage***等函数，而这些函数最终都会调用enqueueMessage函数

```
//Handler.java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    //...
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

可以看到enqueueMessage为msg设置了target字段。



# 同步屏障的工作原理--MessageQueue

**设置了同步屏障之后，Handler只会处理异步消息。再换句话说，同步屏障为Handler消息机制增加了一种简单的优先级机制，异步消息的优先级要高于同步消息**

```
Message next() {
    //...

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        //...

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {//碰到同步屏障
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                // do while循环遍历消息链表
                // 跳出循环时，msg指向离表头最近的一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //...
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        //将msg从消息链表中移除
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    //返回异步消息
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            //...
        }

        //...
    }
}
```

可以看到，当设置了同步屏障之后，next函数将会忽略所有的同步消息，返回异步消息。**换句话说就是，设置了同步屏障之后，Handler只会处理异步消息。再换句话说，同步屏障为Handler消息机制增加了一种简单的优先级机制，异步消息的优先级要高于同步消息。**



## 屏障消息的作用
说完了屏障消息的插入和删除，那么屏障消息在哪里起作用的？它跟前面提到的异步消息又有什么关联呢？我们可以看到MessageQueue的next方法里有这么一段：

```
// 这里就是判断当前消息是否是屏障消息，判断依据就是msg.target==null, 如果存在屏障消息，那么在它之后进来的消息中，
// 只把异步消息放行继续执行，同步消息阻塞，直到屏障消息被remove掉。
if (msg != null && msg.target == null) {
    // Stalled by a barrier.  Find the next asynchronous message in the queue.
    do {
        prevMsg = msg;
        msg = msg.next;
        // 这里的isAsynchronous方法就是前面设置进msg的async参数，通过它判断如果是异步消息，则跳出循环，把该异步消息返回
        // 否则是同步消息，把同步消息阻塞。
    } while (msg != null && !msg.isAsynchronous());
}
```



## 详细图解

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210423143333.png)

当消息队列开启同步屏障的时候（即标识为`msg.target == null`），消息机制会通过循环遍历，优先处理**异步消息**。这样，同步屏障就起到了一种过滤和优先级的作用。

下面用示意图简单说明：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210423143354.png)

 

 

如上图所示，在消息队列中有同步消息和异步消息（黄色部分）以及一道墙----同步屏障（红色部分）。有了内存屏障的存在，msg_2这个异步消息可以被处理，而后面的 msg_3等同步消息不会被处理。那么什么时候这些同步消息可以被处理呢？那就需要移除这个内存屏障，调用removeSyncBarrier()即可。

### 类比

举个例子。开演唱会的时候，观众们都在体育馆门口排队依次等候检票入场（相当于消息队列中的普通消息），这个时候有一大波工作人员来了（相当于异步消息，优先级高于观众），如果他们出示工作证（不出示工作证，就相当于普通观众入场，也还是需要排队，这种情形就是最前面所说的仅仅设置了msg.setAsynchronous(true)），保安立马拦住（出示工作证就拦住就相当于开启了同步屏障）进场的观众，先让工作人员进去（只处理异步消息，而过滤掉同步消息）。等工作人员全部进去了，保安不再阻拦观众（即移除内存屏障），这样观众又可以进场了。只要保安不解除拦截，那么后面的观众就永远不可能进场（不移除内存屏障，同步消息就不会得到处理）

### 总结

- Handler在发消息时，MessageQueue已经对消息**按照了等待时间**进行了排序。
- MessageQueue**不仅包含了Java层消息机制同时包含Native消息机制**
- Handler消息分为**异步消息**与**同步消息**两种。
- MessageQueue中存在**“屏障消息“**的概念，当出现屏障消息时，会执行最近的异步消息，同步消息会被过滤。
- MessageQueue在执行完消息队列中的消息等待更多消息时，**会处理一些空闲任务，如GC操作等**



#  应用


## 屏障消息的实际应用-Choreographer-编舞者
[[4-屏幕刷新机制#Choreographer]]
屏障消息的作用是把在它之后入队的同步消息阻塞，但是异步消息还是正常按顺序取出执行，那么它的实际用途是什么呢？我们看到ViewRootImpl.scheduleTraversals()用到了屏障消息和异步消息。

TraversalRunnable的run(),在这个run()中会执行doTraversal()，最终会触发View的绘制流程：measure(),layout(),draw()。为了让绘制流程尽快被执行，用到了同步屏障技术

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 这里先将主线程的MessageQueue设置了个消息屏障
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 这里发送了个异步消息mTraversalRunnable，这个mTraversalRunnable最终会执行doTraversal(),也就是会触发View的绘制流程
        // 也就是说通过设置屏障消息，会把主线程的同步消息先阻塞，优先执行View绘制这个异步消息进行界面绘制。
        // 这很好理解，界面绘制的任务肯定要优先，否则就会出现界面卡顿。
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
 
private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    if (DEBUG_FRAMES) {
        Log.d(TAG, "PostCallback: type=" + callbackType
                + ", action=" + action + ", token=" + token
                + ", delayMillis=" + delayMillis);
    }
 
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
 
        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            // 设置该消息是异步消息
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

## 我们能用屏障消息做什么？

那么除了系统中使用到了屏障消息，我们在开发中有什么场景能派上用场吗? 运用屏障消息可以阻塞同步消息的特性，我们可以用来实现UI界面初始化和数据加载同时进行。

我们一般在Activity创建的时候，为了减少空指针异常的发生，都会在onCreate先setContent，然后findView初始化控件，然后再执行网络数据加载的异步请求，待网络数据加载完成后，再刷新各个控件的界面。

试想一下，怎么利用屏障消息的特性来达到**界面初始化和异步网络数据**的加载同时进行，而不影响界面渲染？先来看一个时序图：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210423114345.png)



```java

// 在上一个页面里异步加载下一个页面的数据
 // 网络请求返回的数据
    Data netWorkData;
    // 创建屏障消息会生成一个token，这个token用来删除屏障消息，很重要。
    int barrierToken;
 
    // 创建异步线程加载网络数据
    HandlerThread thread = new HandlerThread("preLoad"){
        @Override
        protected void onLooperPrepared() {
            Handler mThreadHandler = new Handler(thread.getLooper());
            // 1、把请求网络耗时消息推入消息队列
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    // 异步耗时操作：网络请求数据，赋值给netWorkData
                    netWorkData = xxx;
 
                }
            });
 
            // 2、然后给异步线程的队列发一个屏障消息推入消息队列
            barrierToken = thread.getLooper().getQueue().postSyncBarrier();
 
            // 3、然后给异步线程的消息队列发一个刷新UI界面的同步消息
            // 这个消息在屏障消息被remove前得不到执行的。
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    // 回调主线程, 把netWorkData赋给监听方法，刷新界面
 
                }
            });
        }
    };
    thread.start();
 
 
// 当前界面初始化界面
protected void onCreate(Bundle savedInstanceState) {
    setContentView(view);
 
    // 各种findview操作完成
    Button btn = findViewById(R.id.xxx);
    ...
    // 4、待控件初始化完成，把异步线程设置的屏障消息remove掉，这样异步线程请求数据完成后，3、处的刷新UI界面的同步消息就有机会执行，就可以安全得刷新界面了。
    thread.getLooper().getQueue().removeSyncBarrier(barrierToken);
}

```


除此之外，在一些第三方库中都有使用IdleHandler，比如LeakCanary，Glide中有使用到。

那么对于我们来说，IdleHandler可以有些什么使用场景呢？根据它最核心的原理，在消息队列空闲的时候做点事情，那么对于主线程来讲，我们有很多的一些代码不是必须要跟随生命周期方法同步执行的，就可以用IdleHandler，减少主线程的耗时，也就减少应用或者Activity的启动时间。例如：一些第三方库的初始化，埋点尤其是延时埋点上报等，都可以用IdleHandler添加到消息队列里。

==好了，提个问题：前面我们说了在主线程创建的main函数里创建了Handler和Looper，回顾了上面的Handler机制的原理，我们都知道一般线程执行完就会退出，由系统回收资源，那Android UI线程也是基于Handler Looper机制的，那么为什么UI线程可以一直常驻？不会被阻塞呢？==

> 因为Looper在执行loop方法里，是一个for循环，也就是说线程永远不会执行完退出，所以打开APP可以一直显示，Activity的生命周期就是通过消息队列把消息一个一个取出来执行的，然后因为MessageQueue的休眠唤醒机制，当消息队列里没有消息时，消息队列会进入休眠，并释放CPU资源，当又有新消息进入队列时，会唤醒队列，把消息取出来执行。



# 面试题

## 同步屏障和异步消息是怎么实现的？

其实在`Handler`机制中，有三种消息类型：

- `同步消息`。也就是普通的消息。
- `异步消息`。通过setAsynchronous(true)设置的消息。
- `同步屏障消息`。通过postSyncBarrier方法添加的消息，特点是target为空，也就是没有对应的handler。

这三者之间的关系如何呢？

- 正常情况下，同步消息和异步消息都是正常被处理，也就是根据时间when来取消息，处理消息。
- 当遇到同步屏障消息的时候，就开始从消息队列里面去找异步消息，找到了再根据时间决定阻塞还是返回消息。



```java
Message msg = mMessages;
if (msg != null && msg.target == null) {
      do {
      prevMsg = msg;
      msg = msg.next;
      } while (msg != null && !msg.isAsynchronous());
}

```

也就是说同步屏障消息不会被返回，他只是一个标志，一个工具，遇到它就代表要去先行处理异步消息了。

所以同步屏障和异步消息的存在的意义就在于有些消息需要`“加急处理”`。

## 同步屏障和异步消息有具体的使用场景吗？

使用场景就很多了，比如绘制方法`scheduleTraversals`。

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // 同步屏障，阻塞所有的同步消息
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 通过 Choreographer 发送绘制任务
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        }
    }

    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
    msg.arg1 = callbackType;
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, dueTime);

复制代码
```

在该方法中加入了同步屏障，后续加入一个异步消息`MSG_DO_SCHEDULE_CALLBACK`，最后会执行到`FrameDisplayEventReceiver`，用于申请VSYNC信号。



## 17、什么是Handler的同步屏障？
答: 同步屏障是一种使得异步消息可以被更快处理的机制


## 18、能不能让一个Message被加急处理？
答：可以，添加加同步屏障，并发送异步消息