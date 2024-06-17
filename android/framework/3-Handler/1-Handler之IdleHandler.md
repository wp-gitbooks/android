# 线索
```java
Looper.myQueue().addIdleHandler(new IdleHandler() {

    @Override
    public boolean queueIdle() {
        ...
        return false;
    }
}
```

queueIdle 为 false 一次性，为 true 每次都会执行。
观察者模式。


# 概述

IdleHandler：**空闲监听器**（就像我没事做了，在群里发了个表情，这时候其他人就知道我很闲了）

在每次next获取消息进行处理时，发现没有可以处理的消息（队列空，只有延时消息并且没到时间，同步阻塞时没有异步消息）都会通知这些订阅者。

适合做一些可有可无的东西，因为这个通知太不稳定了（就像别人说过几天请你吃饭，你要真傻等着你估计能饿死）



通过上面的分析，我们已经知道 IdleHandler 是 MessageQueue 的静态内部接口，是在队列空闲时会执行的，也了解它是如何被触发的，这和消息机制息息相关。

IdleHandler 是 Handler 提供的一种在消息队列空闲时，执行任务的时机。但它执行的时机依赖消息队列的情况，那么如果 MessageQueue 一直有待执行的消息时，IdleHandler 就一直得不到执行，也就是它的执行时机是不可控的，不适合执行一些对时机要求比较高的任务。



## IdleHandler有什么用？(what)
1. IdleHandler 是 Handler 提供的一种在消息队列空闲时，执行任务的时机；
2. 当 MessageQueue 当前没有立即需要处理的消息时，会执行 IdleHandler；


## 怎么使用---MessageQueue#addIdleHandler(IdleHandler handler) (how)

要使用IdleHandler只需要调用**MessageQueue#addIdleHandler(IdleHandler handler)**方法即

```java
Looper.myQueue().addIdleHandler(new IdleHandler() {

    @Override
    public boolean queueIdle() {
        ...
        return false;
    }
}
```


# 源码分析（IdleHandler原理）

## 简单

### 接口

```java
// MessageQueue.java
public static interface IdleHandler {
  boolean queueIdle();
}


// MessageQueue.java
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
 
...
public void addIdleHandler(@NonNull IdleHandler handler) {
	// ...
  synchronized (this) {
    mIdleHandlers.add(handler);
  }
}
public void removeIdleHandler(@NonNull IdleHandler handler) {
  synchronized (this) {
    mIdleHandlers.remove(handler);
  }
}

```


## 详细

注释中很明确地指出当消息队列空闲时会执行IdleHandler的**queueIdle**()方法，该方法返回一个**boolean**值，

如果为**false**则执行完毕之后移除这条消息（一次性完事），

如果为**true**则保留，等到下次空闲时会再次执行（重复执行）

```java
    /**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */

        boolean queueIdle();

    }
```



```java
Message next() {
        ......
        for (;;) {
            ......
            synchronized (this) {
        // 此处为正常消息队列的处理
                ......
                if (mQuitting) {
                    dispose();
                    return null;
                }
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
            }

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
            pendingIdleHandlerCount = 0;
            nextPollTimeoutMillis = 0;
        }
    }
```

大概流程是这样的：

1. 如果本次循环拿到的**Message为空**，或者！这个Message是一个**延时的消息而且还没到指定的触发**时间，那么，就认定当前的队列为空闲状态，
2. 接着就会遍历mPendingIdleHandlers数组(这个数组里面的元素每次都会到mIdleHandlers中去拿)来调用每一个IdleHandler实例的queueIdle方法，
3. 如果这个方法返回false的话，那么这个实例就会从mIdleHandlers中移除，也就是当下次队列空闲的时候，不会继续回调它的queueIdle方法了。

### IdleHandler返回true和false带来的不同结果

**queueIdle() 方法如果返回 false**，那么这个 IdleHandler 只会**执行一次**，执行完后会从队列中删除，如果返回 true，那么执行完后不会被删除，只要执行 MessageQueue.next 时消息队列中没有可执行的消息，即为空闲时间，那么 IdleHandler 队列中的 IdleHandler 还会继续被执行。

```java
public void clickTestIdleHandler(View view) {
    // 添加第一个 IdleHandler
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
        @Override
        public boolean queueIdle() {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.e("Test","IdleHandler1 queueIdle");
            return false;
        }
    });
    // 添加第二个 IdleHandler
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
        @Override
        public boolean queueIdle() {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.e("Test","IdleHandler2 queueIdle");
            return false;
        }
    });
    Handler handler = new Handler();
    // 添加第一个Message
    Message message1 = Message.obtain(handler, new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.e("Test","message1");
        }
    });
    message1.sendToTarget();
    // 添加第二个Message
    Message message2 = Message.obtain(handler, new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.e("Test","message2");
        }
    });
    message2.sendToTarget();
}
```



上面的例子中分别向消息队列中添加来两个 IdleHandler 和两个 Message，其中 IdleHandler 的 queueIdle() 方法返回 false，下面来看一下结果：

```
16:23:07.726 E/Test: message1
16:23:09.727 E/Test: message2
16:23:11.732 E/Test: IdleHandler1 queueIdle
16:23:13.733 E/Test: IdleHandler2 queueIdle
```

可以看到 IdleHandler 是在消息队列的其它消息执行完后才执行的，而且只执行来一次。
再来看一下 queueIdle() 方法返回 true 的情况：

```
16:27:02.083  E/Test: message1
16:27:04.084  E/Test: message2
16:27:06.090  E/Test: IdleHandler1 queueIdle
16:27:08.090  E/Test: IdleHandler2 queueIdle
16:27:10.095  E/Test: IdleHandler1 queueIdle
16:27:12.096  E/Test: IdleHandler2 queueIdle
16:27:14.099  E/Test: IdleHandler1 queueIdle
16:27:16.099  E/Test: IdleHandler2 queueIdle
16:27:43.788  E/Test: IdleHandler1 queueIdle
16:27:45.788  E/Test: IdleHandler2 queueIdle
16:27:47.792  E/Test: IdleHandler1 queueIdle
16:27:49.793  E/Test: IdleHandler2 queueIdle
```



可以看到，当 queueIdle() 方法返回 true时会多次执行，即 IdleHandler 执行一次后不会从 IdleHandler 的队列中删除，等下次空闲时间到来时还会继续执行


## IdleHander是如何保证不进入死循环的？

当队列空闲时，会循环执行一遍 `mIdleHandlers` 数组并执行 `IdleHandler.queueIdle()` 方法。而如果数组中有一些 IdleHander 的 `queueIdle()` 返回了 `true`，则会保留在 `mIdleHanders` 数组中，下次依然会再执行一遍。

注意现在代码逻辑还在 `MessageQueue.next()` 的循环中，在这个场景下 IdleHandler 机制是如何保证不会进入死循环的？

有些文章会说 IdleHandler 不会死循环，是因为下次循环调用了 `nativePollOnce()` 借助 epoll 机制进入休眠状态，下次有新消息入队的时候会重新唤醒，但这是不对的。

注意看前面 `next()` 中的代码，在方法的末尾会重置 pendingIdleHandlerCount 和 nextPollTimeoutMillis。

```java
Message next() {
	// ...
  int pendingIdleHandlerCount = -1; 
  int nextPollTimeoutMillis = 0;
  for (;;) {
		nativePollOnce(ptr, nextPollTimeoutMillis);
    // ...
    // 循环执行 mIdleHandlers
    // ...
    pendingIdleHandlerCount = 0;
    nextPollTimeoutMillis = 0;
  }
}

```

nextPollTimeoutMillis 决定了下次进入 `nativePollOnce()` 超时的时间，它传递 0 的时候等于不会进入休眠，所以说 `natievPollOnce()` 进入休眠所以不会死循环是不对的。

这很好理解，毕竟 `IdleHandler.queueIdle()` 运行在主线程，它执行的时间是不可控的，那么 MessageQueue 中的消息情况可能会变化，所以需要再处理一遍。

**实际不会死循环的关键是在于 pendingIdleHandlerCount**，我们看看下面的代码。

```java
Message next() {
	// ...
  // Step 1
  int pendingIdleHandlerCount = -1; 
  int nextPollTimeoutMillis = 0;
  for (;;) {
    nativePollOnce(ptr, nextPollTimeoutMillis);

    synchronized (this) {
      // ...
      // Step 2
      if (pendingIdleHandlerCount < 0
          && (mMessages == null || now < mMessages.when)) {
        pendingIdleHandlerCount = mIdleHandlers.size();
      }
     	// Step 3
      if (pendingIdleHandlerCount <= 0) {
        mBlocked = true;
        continue;
      }
      // ...
    }
		// Step 4
    pendingIdleHandlerCount = 0;
    nextPollTimeoutMillis = 0;
  }
}

```

我们梳理一下：

- Step 1，循环开始前，`pendingIdleHandlerCount` 的初始值为 -1；
- Step 2，在 `pendingIdleHandlerCount<0` 时，才会通过 `mIdleHandlers.size()` 赋值。也就是说只有第一次循环才会改变 `pendingIdleHandlerCount` 的值；
- Step 3，如果 `pendingIdleHandlerCount<=0` 时，则循环 continus；
- Step 4，重置 `pendingIdleHandlerCount` 为 0；

在第二次循环时，`pendingIdleHandlerCount` 等于 0，在 Step 2 不会改变它的值，那么在 Step 3 中会直接 continus 继续下一次循环，此时没有机会修改 `nextPollTimeoutMillis`。

那么 `nextPollTimeoutMillis` 有两种可能：-1 或者下次唤醒的等待间隔时间，在执行到 `nativePollOnce()` 时就会进入休眠，等待再次被唤醒。

下次唤醒时，`mMessage` 必然会有一个待执行的 Message，则 `MessageQueue.next()` 返回到 `Looper.loop()` 的循环中，分发处理这个 Message，之后又是一轮新的 `next()` 中去循环。


## 注意事项

关于IdleHandler的使用还有一些注意事项，我们也需要注意：

1. MessageQueue 提供了add/remove IdleHandler方法，但是我们**不一定需要成对使用它们**，因为IdleHandler.queueIdle() 的返回值返回 false 的时候可以移除 IdleHanlder。
2. 不要将一些不重要的启动服务放到 IdleHandler 中去管理，因为它的处理时机不可控，如果 MessageQueue 一直有待处理的消息，那么它的执行时机会很靠后。
3. 当 mIdleHanders 一直不为空时，为什么不会进入死循环？ 只有在 pendingIdleHandlerCount 为 -1 时，才会尝试执行 mIdleHander； pendingIdlehanderCount 在 next() 中初始时为 -1，执行一遍后被置为 0，所以不会重复执行；

## 工作原理

1、观察者模式

2、一直不为空为什么不会进入死循环？



# 应用

## 来看看它在源码里的使用场景：
比如在ActivityThread中，就有一个名叫GcIdler的内部类，实现了IdleHandler接口。

它在queueIdle方法被回调时，会做强行GC的操作（即调用BinderInternal的forceGc方法），但强行GC的前提是，与上一次强行GC至少相隔5秒以上。 

### ActivityThread中GcIdler
那么我们就看看最为明显的ActivityThread中声明的GcIdler。在ActivityThread中的**H**收到**GC_WHEN_IDLE**消息后，会执行scheduleGcIdler，将GcIdler添加到MessageQueue中的空闲任务集合中。具体如下：

```java
 void scheduleGcIdler() {
        if (!mGcIdlerScheduled) {
            mGcIdlerScheduled = true;
            //添加GC任务
            Looper.myQueue().addIdleHandler(mGcIdler);
        }
        mH.removeMessages(H.GC_WHEN_IDLE);
    }
```

ActivityThread中GcIdler的详细声明：

```java
   //GC任务
   final class GcIdler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            doGcIfNeeded();
            //执行后，就直接删除
            return false;
        }
    }
    // 判断是否需要执行垃圾回收。
    void doGcIfNeeded() {
        mGcIdlerScheduled = false;
        final long now = SystemClock.uptimeMillis();
        //获取上次GC的时间
        if ((BinderInternal.getLastGcTime()+MIN_TIME_BETWEEN_GCS) < now) {
            //Slog.i(TAG, "**** WE DO, WE DO WANT TO GC!");
            BinderInternal.forceGc("bg");
        }
    }
```

GcIdler方法理解起来很简单、就是获取上次GC的时间，判断是否需要GC操作。如果需要则进行GC操作。这里ActivityThread中还声明了其他空闲时的任务。如果大家对其他空闲任务感兴趣，可以自行研究。

#### 那这个GcIdler会在什么时候使用呢？

当ActivityThread的mH(Handler)收到GC_WHEN_IDLE消息之后。

#### 何时会收到GC_WHEN_IDLE消息？

当AMS(ActivityManagerService)中的这两个方法被调用之后：

- doLowMemReportIfNeededLocked， 这个方法看名字就知道是不够内存的时候调用的了。
- activityIdle， 这个方法呢，就是当ActivityThread的handleResumeActivity方法被调用时(Activity的onResume方法也是在这方法里回调)调用的。



### ActivityThread.Idle
知道了这个 IdleHandler 是如何被触发的，我们再来看看系统源码时如何使用它的：比如 ActivityThread.Idle 在 ActivityThread.handleResumeActivity()中调用。

```java
@Override
3808    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
3809            String reason) {
3810        // If we are getting ready to gc after going to the background, well
3811        // we are back active so skip it.
3812        unscheduleGcIdler();
3813        mSomeActivitiesChanged = true;
						···
  					//该方法最终会执行 onResume方法
3816        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
3817        if (r == null) {
3818            // We didn't actually resume the activity, so skipping any follow-up actions.
3819            return;
3820        }
						··· 
  					···
3921
3922        r.nextIdle = mNewActivities;
3923        mNewActivities = r;
3924        if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
3925        Looper.myQueue().addIdleHandler(new Idler());
3926    }
```

可以看到在 handleResumeActivity() 方法中末尾会执行 Looper.myQueue().addIdleHandler(new Idler())，也就是说在 onResume 等方法都执行完，界面已经显示出来了，那么这个 Idler 是用来干嘛的呢？先来看看这个内部类的定义：

```java
1829    private class Idler implements MessageQueue.IdleHandler {
1830        @Override
1831        public final boolean queueIdle() {
1832            ActivityClientRecord a = mNewActivities;
								···
1838            if (a != null) {
1839                mNewActivities = null;
1840                IActivityManager am = ActivityManager.getService();
1841                ActivityClientRecord prev;
1842                do {
  											//打印一些日志
1843                    if (localLOGV) Slog.v(
1844                        TAG, "Reporting idle of " + a +
1845                        " finished=" +
1846                        (a.activity != null && a.activity.mFinished));
1847                    if (a.activity != null && !a.activity.mFinished) {
1848                        try {
  															//AMS 进行一些资源的回收
1849                            am.activityIdle(a.token, a.createdConfig, stopProfiling);
1850                            a.createdConfig = null;
1851                        } catch (RemoteException ex) {
1852                            throw ex.rethrowFromSystemServer();
1853                        }
1854                    }
1855                    prev = a;
1856                    a = a.nextIdle;
1857                    prev.nextIdle = null;
1858                } while (a != null);
1859            }
1860            if (stopProfiling) {
1861                mProfiler.stopProfiling();
1862            }
								//确认Jit 可以使用，否则抛出异常
1863            ensureJitEnabled();
1864            return false;
1865        }
1866    }
复制代码
```

可以看到在 queueIdle 方法中会进行回收等操作，下面会详细讲解，但这一些都是等 onResume 方法执行完，界面已经显示这些更重要的事情已经处理完了，空闲的时候开始处理这些事情。也就是说系统的设计逻辑是保障最重要的逻辑先执行完，再去处理其他次要的事情。

但是如果 MessageQueue 队列中一直有消息，那么 IdleHandler 就一直没有机会被执行，那么原本该销毁的界面的 onStop，onDestory 就得不到执行吗？不是这样的，在 resumeTopActivityInnerLocked() -> completeResumeLocked() -> scheduleIdleTimeoutLocked() 方法中会发送一个会发送一个延迟消息（10s），如果界面很久没有关闭(如果界面需要关闭)，那么 10s 后该消息被触发就会关闭界面，执行 onStop 等方法


## 其他可以想到的场景

1.Activity启动优化：onCreate，onStart，onResume中耗时较短但非必要的代码可以放到IdleHandler中执行，减少启动时间

2.想要在一个View绘制完成之后添加其他依赖于这个View的View，当然这个用View#post()也能实现，区别就是前者会在消息队列空闲时执行

3.发送一个返回true的IdleHandler，在里面让某个View不停闪烁，这样当用户发呆时就可以诱导用户点击这个View，这也是种很酷的操作

4.一些第三方库中有使用，比如LeakCanary，Glide中有使用到，具体可以自行去查看


### LeakCanary
[[LeakCanary]]
我们来看看 LeakCanary 的使用，是在AndroidWatchExecutor 这个类中
```java
public final class AndroidWatchExecutor implements WatchExecutor {

  static final String LEAK_CANARY_THREAD_NAME = "LeakCanary-Heap-Dump";
  private final Handler mainHandler;
  private final Handler backgroundHandler;
  private final long initialDelayMillis;
  private final long maxBackoffFactor;

  public AndroidWatchExecutor(long initialDelayMillis) {
    mainHandler = new Handler(Looper.getMainLooper());
    HandlerThread handlerThread = new HandlerThread(LEAK_CANARY_THREAD_NAME);
    handlerThread.start();
    backgroundHandler = new Handler(handlerThread.getLooper());
    this.initialDelayMillis = initialDelayMillis;
    maxBackoffFactor = Long.MAX_VALUE / initialDelayMillis;
  }
	//初始调用
  @Override public void execute(Retryable retryable) {
    // 最终都会切换到主线程，调用waitForIdle()方法
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      waitForIdle(retryable, 0);
    } else {
      postWaitForIdle(retryable, 0);
    }
  }

  private void postWaitForIdle(final Retryable retryable, final int failedAttempts) {
    mainHandler.post(new Runnable() {
      @Override public void run() {
        waitForIdle(retryable, failedAttempts);
      }
    });
  }
	//IdleHandler 使用
  void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }
  //最终的调用方法
	private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    backgroundHandler.postDelayed(new Runnable() {
      @Override public void run() {
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
  }
}
```

再来看看 execute() 这个方法在何处被调用，我们知道 LeakCancary 是在界面销毁 onDestroy 方法中进行 refWatch.watch() 的，而watch() -> ensureGoneAsync() -> execute()

```java
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```

而 ensureGone() 中会进行 GC 回收和一些分析等操作，所以通过这些分析后，**我们可以知道 LeakCanary 进行内存泄漏检测并不是 onDestry 方法执行完成后就进行垃圾回收和一些分析的，而是利用 IdleHandler 在空闲的时候进行这些操作的，尽量不去影响主线程的操作**


### 提供一个android没有的声明周期回调时机

如果有这种需求，想要在某个activity绘制完成去做一些事情，那这个时机是什么时候呢？有同学可能觉得onResume()是一个合适的机会，不是可是这个onResume() 真的是各种绘制都已经完成才回调的吗？No, too naive  ~~

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160504.png)



你看谷老师说了，onStart是用户可见，onResume是用户可交互，谷老师可没说onResume是绘制完成吧~那么android那些耗时的measure, layout, draw是在什么时候执行的呢？它们跟onResume()又有何关系呢？让我们先来看看源码吧~


**1. ActivityThread.java**

我们知道app的进程其实是ActivityThread, 那么activity的生命周期自然是它来执行了，

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160524.png)


performResumeActivity就是回调onResume了， 我们继续看wm.addView方法， 这个ViewManager是一个接口，其实现者是WindowManagerImpl


**2.WindowManagerImpl.java**

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160539.png)



这个mGlobal是WindowManagerGlobal对象，我们继续



**3.WindowManagerGlobal.java**

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160550.png)



这里我们new 出了ViewRootImpl对象， 我们知道这个对象就是android view的根对象了，负责view绘制的measure, layout, draw的巨长的方法 performTraversals就是这个类的，我们继续看setView方法



**4.ViewRootImpl.java**

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160601.png)



这个函数调用了关键方法requestLayout(), 我们继续跟踪，顺便说下，后面一连串的BadTokenException就是我们常常遇到的dialog相关抛出的，也有些特殊场景也会出这个异常，可以到这里查看线索。

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160611.png)



 调用了scheduleTraversals， 从名字就能看出来了吧：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160621.png)



它往Choreographer里面post了一个runnable， 这个Choreographer是android负责帧率刷新相关的东西，我们暂时可以不关注它，可以理解为往主线程post一个消息是一样的，顺便说下这个Choreographer可以做帧率检测相关的东西，，可以用于卡顿检测什么的···

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160632.png)

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160640.png)



 我们看这个runnable果然是去执行了那个巨长无比的函数performTraversals函数, 现在我们可以总结下流程了：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160651.jpg)





**结论：****所以如果我们想在界面绘制出来后做点什么，那么在onResume里面显然是不合适的，它先于measure等流程了**， **有人可能会说在onResume里面post一个runnable可以吗？还是不行，因为那样就会变成这个样子**

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160700.jpg)
所以你的行为一样会在绘制之前执行，这个时候我们的主角IdleHandler就发挥作用了，我们前面说了，它是在looper里面message暂时执行完毕了就会回调，顾名思义嘛，Idle就是队列为空的意思，那么我们的onResume和measure, layout, draw都是一个个message的话，这个IdleHandler就提供了一个它们都执行完毕的回调了，大概就是这样

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160715.jpg)

说了这么多，那么现在获取到这个时机有什么用呢？ look！！

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160725.gif)



这个是我们地图的公交详情页面， 进入之后产品要求左边的页卡需要展示，可以看到左边的页卡是一个非常复杂的布局，那么进入之后的效果可以明显看到头部的展示信息是先显示空白再100毫秒左右之后才展示出来的，原因就是这个页卡的内容比较复杂，用数据向它填充的时候花了较长时间，代码如下：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160754.png)

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160831.png)



可以看到这个detailView就是这个侧滑的页卡了，填充里面的数据花了90ms，如果这个时间是用在了界面view绘制之前的话，就会出现以上的效果了，view先是白的，再出现，这样就体验不好了，如果我们把它放到IdleHandler里面呢？代码如下：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160440.png)



效果是这样的：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160442.gif)



看出不同了吗？顶部的页卡先展示出来了，这样体验是不是会更好一些呢。虽然只有短短90ms，不过我们做app也应该关注这种细节优化的，是吧~ 这个做法也提供了一种思路，android本身提供的activity框架和fragment框架并没有提供绘制完成的回调，如果我们自己实现一个框架，就可以使用这个IdleHandler来实现一个onRenderFinished这种回调了。



### 可以结合HandlerThread, 用于单线程消息通知器

我们先思考一个问题，如果有一个model数据管理模块，怎么设计？比如地图的收藏模块的model部分。就是下面这个图的小星星：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423161025.jpg)



它原来的model设计大概是这个样子的：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423161036.png)



 由于这个model是单例的，而且是多线程可以访问的，所以它的增删改查都加上了锁，而且由于外部访问需要遍历有哪些收藏点，所以外部遍历列表也需要加锁，大概是这样的：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423161046.png)



 因为是多线程可访问的，如果遍历不加锁的话，其他线程删除了一个收藏，就会crash的，**原来的这样设计有几个不好的地方：**

**1. 外部使用者需要关系锁的使用，增加了负担，不用还不安全**

**2. 如果在主线程加锁的话，可能另一个线程执行操作会阻塞主线程造成anr**



总之，多线程代码就是容易出错，而且真的出错的时候查起来太费劲了，目前收藏夹模块就有N多bug，所以我想用单线程来解决这个问题，由于model层的访问需要数据库和网络等，所以需要异步线程，那么单线程队列+异步线程，首先想到的就是HandlerThread, 大概架构如下：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423161101.png)



 现在，我们把原来多线程的逻辑改到了单线程里面，各种收藏的model共用一个HandlerThread，这样我们增删改查都不用加锁了，出错几率大大减小，而且这种model的设计有点类似插件的意思，可以很方便的增加其他收藏。



Ok， 那么跟我们的主题IdleHandler有什么关系呢？思考这样一个问题，地图上的小星星需要实时更新，也就是model的任何变化都需要显示到地图上，那么收藏的小星星就应该作为model的观察者，以前的做法是向收藏model注册监听，在每一个增删改查操作后都对观察者回调，大概是这样：

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423161113.png)



这样有一个小小的问题，就是如果有一个操作生成10个快速连续的增删改查操作，那么我们的UI就会收到10次回调，而这种场景下我们其实只需要最后一次回调就够了，中间操作其实不用刷新UI的。



那么现在改成单线程模型，我们又该如何处理这个问题呢？当然我们也能在每个post到异步线程的runnable里面去回调观察者，但这样未免不够优雅，所以这个时候IdleHandler不就又可以发挥作用了吗？它是在消息暂时处理完的时候回调的呀，不是很符合我们的时机么，对吧？

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423160933.png)



就是这个样子了，这里为什么不用第一个场景下的Looper.myQueue().addIdleHandler()呢？注意这个地方Looper.myQueue()如果在主线程调用就会使用主线程looper了，所以我选择反射这个HandlerThread的looper来设置它，这个IdleHandler我们返回了true, 表示我们要长期监听消息队列，因为返回false，下次就没有回调了哦。



好了，结论是这个地方IdleHandler用作了一个消息的触发器，是不是挺有意思的呢？





# 面试题

## Q：IdleHandler 有什么用？

1. IdleHandler 是 Handler 提供的一种在消息队列空闲时，执行任务的时机；
2. 当 MessageQueue 当前没有立即需要处理的消息时，会执行 IdleHandler；

## Q：MessageQueue 提供了 add/remove IdleHandler 的方法，是否需要成对使用？

1. 不是必须；
2. IdleHandler.queueIdle() 的返回值，可以移除加入 MessageQueue 的 IdleHandler；

## Q：当 mIdleHanders 一直不为空时，为什么不会进入死循环？

1. 只有在 pendingIdleHandlerCount 为 -1 时，才会尝试执行 mIdleHander；
2. pendingIdlehanderCount 在 next() 中初始时为 -1，执行一遍后被置为 0，所以不会重复执行；

## Q：是否可以将一些不重要的启动服务，搬移到 IdleHandler 中去处理？

1. 不建议；
2. IdleHandler 的处理时机不可控，如果 MessageQueue 一直有待处理的消息，那么 IdleHander 的执行时机会很靠后；

## Q：IdleHandler 的 queueIdle() 运行在那个线程？

1. 陷进问题，queueIdle() 运行的线程，只和当前 MessageQueue 的 Looper 所在的线程有关；
2. 子线程一样可以构造 Looper，并添加 IdleHandler；


## framework 中如何使用 IdleHander？

到这里基本上就讲清楚 IdleHandler 如何使用以及一些细节，接下来我们来看看，在系统中，有哪些地方会用到 IdleHandler 机制。

在 AS 中搜索一下 IdleHandler。



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210423162259)



简单解释一下：

1. ActivityThread.Idler 在 `ActivityThread.handleResumeActivity()` 中调用。
2. ActivityThread.GcIdler 是在内存不足时，强行 GC；
3. Instrumentation.ActivityGoing 在 Activity onCreate() 执行前添加；
4. Instrumentation.Idler 调用的时机就比较多了，是键盘相关的调用；
5. TextToSpeechService.SynthThread 是在 TTS 合成完成之后发送广播；

有兴趣可以自己追一下源码，这些都是使用的场景，具体用 IdleHander 干什么，还是要看业务




## 当 mIdleHanders 一直不为空时，为什么不会进入死循环？

1. 只有在 pendingIdleHandlerCount 为 -1 时，才会尝试执行 mIdleHander；
2. pendingIdlehanderCount 在 next() 中初始时为 -1，执行一遍后被置为 0，所以不会重复执行；



## 既然 IdleHandler 主要是在 MessageQueue 出现空闲的时候被执行，那么何时出现空闲？

MessageQueue 是一个基于消息触发时间的优先级队列，所以**队列出现空闲**存在两种场景。

1. MessageQueue 为空，**没有消息**；
2. MessageQueue 中最近需要处理的消息，是一个**延迟消息（when>currentTime）**，需要滞后执行；

这两个场景，都会尝试执行 IdleHandler

```
Message next() {
	// ...
  int pendingIdleHandlerCount = -1; 
  int nextPollTimeoutMillis = 0;
  for (;;) {
    nativePollOnce(ptr, nextPollTimeoutMillis);

    synchronized (this) {
      // ...
      if (msg != null) {
        if (now < msg.when) {
          // 计算休眠的时间
          nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
        } else {
          // Other code
          // 找到消息处理后返回
          return msg;
        }
      } else {
        // 没有更多的消息
        nextPollTimeoutMillis = -1;
      }
      
      if (pendingIdleHandlerCount < 0
          && (mMessages == null || now < mMessages.when)) {
        pendingIdleHandlerCount = mIdleHandlers.size();
      }
      if (pendingIdleHandlerCount <= 0) {
        mBlocked = true;
        continue;
      }

      if (mPendingIdleHandlers == null) {
        mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
      }
      mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
    }

    for (int i = 0; i < pendingIdleHandlerCount; i++) {
      final IdleHandler idler = mPendingIdleHandlers[i];
      mPendingIdleHandlers[i] = null; 

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

    pendingIdleHandlerCount = 0;
    nextPollTimeoutMillis = 0;
  }
}
```

我们先解释一下 `next()` 中关于 IdleHandler 执行的主逻辑：

1. 准备执行 IdleHandler 时，说明当前待执行的消息为 null，或者这条消息的执行时间未到；
2. 当 `pendingIdleHandlerCount < 0` 时，根据 `mIdleHandlers.size()` 赋值给 `pendingIdleHandlerCount`，它是后期循环的基础；
3. 将 `mIdleHandlers` 中的 IdleHandler 拷贝到 `mPendingIdleHandlers` 数组中，这个数组是临时的，之后进入 for 循环；
4. 循环中从数组中取出 IdleHandler，并调用其 `queueIdle()` 记录返回值存到 `keep` 中；
5. 当 `keep` 为 false 时，从 `mIdleHandler` 中移除当前循环的 IdleHandler，反之则保留；

可以看到 IdleHandler 机制中，最核心的就是在 `next()` 中，当队列空闲的时候，循环 mIdleHandler 中记录的 IdleHandler 对象，如果其 `queueIdle()` 返回值为 `false` 时，将其从 `mIdleHander` 中移除。

需要注意的是，对 `mIdleHandler` 这个 List 的所有操作，都通过 synchronized 来保证线程安全，这一点无需担心





## 是否可以将一些不重要的启动服务，搬移到 IdleHandler 中去处理？

1. 不建议；
2. IdleHandler 的处理时机不可控，如果 MessageQueue 一直有待处理的消息，那么 IdleHander 的执行时机会很靠后；



## IdleHandler 的 queueIdle() 运行在那个线程？

1. 陷进问题，queueIdle() 运行的线程，只和当前 MessageQueue 的 Looper 所在的线程有关；
2. 子线程一样可以构造 Looper，并添加 IdleHandler；


## IdleHandler是什么？怎么使用，能解决什么问题？



## handler空闲会不会阻塞主线程，IdleHandler 使用场景



## Handler机制整体流程；Looper.loop()为什么不会阻塞主线程；IdHandler(闲时机制）；postDelay()的具体实现；post()与sendMessage()区别；使用Handler需要注意什么问题，怎么解决的?





## IdleHandler了解么？合适调用？如何使用？靠谱么？





## 22、IdleHandler是啥？有什么使用场景？

之前说过，当`MessageQueue`没有消息的时候，就会阻塞在next方法中，其实在阻塞之前，`MessageQueue`还会做一件事，就是检查是否存在`IdleHandler`，如果有，就会去执行它的`queueIdle`方法。



```java
    private IdleHandler[] mPendingIdleHandlers;

    Message next() {
        int pendingIdleHandlerCount = -1;
        for (;;) {
            synchronized (this) {
                //当消息执行完毕，就设置pendingIdleHandlerCount
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }

                //初始化mPendingIdleHandlers
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                //mIdleHandlers转为数组
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // 遍历数组，处理每个IdleHandler
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                //如果queueIdle方法返回false，则处理完就删除这个IdleHandler
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;
        }
    }
复制代码
```

当没有消息处理的时候，就会去处理这个`mIdleHandlers`集合里面的每个`IdleHandler`对象，并调用其`queueIdle`方法。 最后根据`queueIdle`返回值判断是否用完删除当前的`IdleHandler`。

然后看看`IdleHandler`是怎么加进去的：



```java
Looper.myQueue().addIdleHandler(new IdleHandler() {  
    @Override  
    public boolean queueIdle() {  
        //做事情
        return false;    
    }  
});

    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }
复制代码
```

ok，综上所述，`IdleHandler`就是当消息队列里面没有当前要处理的消息了，需要堵塞之前，可以做一些空闲任务的处理。

常见的使用场景有：`启动优化`。

我们一般会把一些事件（比如界面view的绘制、赋值）放到`onCreate`方法或者`onResume`方法中。 但是这两个方法其实都是在界面绘制之前调用的，也就是说一定程度上这两个方法的耗时会影响到启动时间。

所以我们可以把一些操作放到`IdleHandler`中，也就是界面绘制完成之后才去调用，这样就能减少启动时间了。

但是，这里需要注意下可能会有坑。

如果使用不当，`IdleHandler`会一直不执行，比如在`View的onDraw方法`里面无限制的直接或者间接调用`View的invalidate方法`。

其原因就在于onDraw方法中执行`invalidate`，会添加一个同步屏障消息，在等到异步消息之前，会阻塞在next方法，而等到`FrameDisplayEventReceiver`异步任务之后又会执行onDraw方法，从而无限循环。



作者：下饭小当家
链接：https://www.jianshu.com/p/4a58568661e8
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



## IdleHandler

见名知意，idle是空闲的意思，那么IdleHandler就是空闲的Handler，有点这个意思，实际上它是MessageQueue中有一个静态接口

```java
public static interface IdleHandler {
    boolean queueIdle();
}
```

可以看到它是一个单方法的接口，也可称为函数型接口，它的作用是：**在UI线程处理完所有View事务后，回调一些额外的操作，且不会堵塞主进程；**我们来实际操作一下



```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        new Handler().getLooper().getQueue().addIdleHandler(new MessageQueue.IdleHandler() {
            @Override
            public boolean queueIdle() {
                Log.d("print", "queueIdle: 空闲时做一些轻量级别");
                return false;
            }
        });
    }
}

//上面代码会打印如下结果
queueIdle: 空闲时做一些轻量级别
```

接着进行源码分析，我们在看下`addIdleHandler`这个方法：



```java
public void addIdleHandler(@NonNull IdleHandler handler) {
    if (handler == null) {
        throw new NullPointerException("Can't add a null IdleHandler");
    }
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}
```

可以看到，被添加进来的handler放到了mIdleHandlers，跟过去看下mIdleHandlers，会发现MessageQueue中定义了IdleHandler的集合和数组，并且有一些操作方法，如下：


```java
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
private IdleHandler[] mPendingIdleHandlers;

public void addIdleHandler(@NonNull IdleHandler handler) {
    if (handler == null) {
        throw new NullPointerException("Can't add a null IdleHandler");
    }
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}

public void removeIdleHandler(@NonNull IdleHandler handler) {
    synchronized (this) {
        mIdleHandlers.remove(handler);
    }
}
```

最后在看下MessageQueue中的`Next`方法，仅贴出关键代码：



```java
Message next() {
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    for (;;) {
        //...
        synchronized (this) {
             //...
             //当前无消息，或还需要等待一段时间消息才能分发，获得IdleHandler的数量
             if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
                 pendingIdleHandlerCount = mIdleHandlers.size();
             }
          
             if (pendingIdleHandlerCount <= 0) {
                 // No idle handlers to run.  Loop and wait some more.
                 //如果没有idle handler需要执行，阻塞线程进入下次循环
                 mBlocked = true;
                 continue;
             }
         //初始化mPendingIdleHandlers
             if (mPendingIdleHandlers == null) {
                 mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
             }
             //把List转化成数组类型
             mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
         }
      
        //循环遍历所有的IdleHandler
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                //获得idler.queueIdle的返回值
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            //keep即idler.queueIdle的返回值，如果为false表明只要执行一次，并移除，否则不移除
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
         }
         // Reset the idle handler count to 0 so we do not run them again.
         //将pendingIdleHandlerCount置为0避免下次再次执行
         pendingIdleHandlerCount = 0;
      
         // 当在执行IdleHandler的时候，可能有新的消息已经进来了
         // 所以这个时候不能阻塞，要回去循环一次看一下
         nextPollTimeoutMillis = 0;
    }
}
```

上述代码解析：

1、当调用next方法的时候，会将pendingIdleHandlerCount赋值为-1

2、判断pendingIdleHandlerCount是否小于0并且MessageQueue 是否为空或者有延迟消息需要执行，如果是则把存储IdleHandler的list的长度赋值给pendingIdleHandlerCount

3、判断如果没有IdleHandler需要执行，阻塞线程进入下次循环，如果有，则初始化mPendingIdleHandlers，把list中的所有IdleHandler放到数组中。这一步是为了不让在执行IdleHandler的时候List被插入新的IdleHandler，造成逻辑混乱

4、循环遍历所有的IdleHandler并执行，查看idler.queueIdle方法的返回值，为false表明这个IdleHandler只需要执行一次，并移除，为true，则不移除

5、将pendingIdleHandlerCount置为0避免下次再次执行， 当在执行IdleHandler的时候，可能有新的消息已经进来了，所以这个时候不能阻塞，要回去循环一次看一下

到这里同步屏障和IdleHandler都讲完了，建议读者配合完整的源码在去仔细阅读一次。

**实际应用**: 可以在IdleHandler里面获取View的宽高



## 19、什么是IdleHandler？

答: IdleHandler是MessageQueue中一个静态函数型接口，它在主线程执行完所有的View事务后，回调一些额外的操作，且不会阻塞主线程