

# 简介
LeakCanary是一个Android和Java的内存泄漏检测库，可以大幅度减少了开发中遇到的OOM问题。


# 使用

2.0以下版本

```java
public class ExampleApplication extends Application {

    @Override 
    public void onCreate() {
        super.onCreate();
        // 如果是在HeapAnalyzer进程里，则返回，因为该进程是专门用来堆内存分析的。
        if (LeakCanary.isInAnalyzerProcess(this)) {
            // This process is dedicated to LeakCanary for heap analysis.
            // You should not init your app in this process.
            return;
        }
        //调用LeakCanary.install()的方法来进行必要的初始化工作，来监听内存泄漏。
        LeakCanary.install(this);
        // Normal app init code...
    }
}
```


# LeakCanary源码分析

## 组成结构

ActivityRefWatch、RefWatcher、RefWatcherBuilder、AndroidRefWatcherBuilder



WatchExecutor、AndroidWatchExecutor


![image-20210624104707366](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210624104707.png)



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210624104800)


## 流程

![LeakCanary注册流程](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210624105147)

![image-20210416100120875](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100120.png)



![image-20210416100040423](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100040.png)




## 原理

https://square.github.io/leakcanary/fundamentals-how-leakcanary-works/

1. Detecting retained objects. 发现保留对象
2. Dumping the heap.    dumping当前堆栈信息
3. Analyzing the heap.     分析堆栈信息
4. Categorizing leaks.      内存泄露情况分类


### 核心原理
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


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416100548.png)



### 其他

#### 基础知识——弱引用 WeakReference 和 引用队列 ReferenceQueue

关于引用类型和引用队列相关知识，读者可以参考[白话 JVM——深入对象引用](https://juejin.im/entry/6844903440682844174)，这篇文章我认为讲解的比较清晰。

这里，我简单举个例子，弱引用在定义的时候可以指定引用对象和一个 ReferenceQueue，弱引用对象在垃圾回收器执行回收方法时，如果原对象只有这个弱引用对象引用着，那么会回收原对象，并将弱引用对象加入到 ReferenceQueue，通过 ReferenceQueue 的 poll 方法，可以取出这个弱引用对象，获取弱引用对象本身的一些信息。看下面这个例子。

```java
mReferenceQueue = new ReferenceQueue<>();
// 定义一个对象
o = new Object();
// 定义一个弱引用对象引用 o,并指定引用队列为 mReferenceQueue
weakReference = new WeakReference<Object>(o, mReferenceQueue);
// 去掉强引用
o = null;
// 触发应用进行垃圾回收
Runtime.getRuntime().gc();
// hack: 延时100ms,等待gc完成
try {
    Thread.sleep(100);
} catch (InterruptedException e) {
    e.printStackTrace();
}
Reference ref = null;
// 遍历 mReferenceQueue，取出所有弱引用
while ((ref = mReferenceQueue.poll()) != null) {
    System.out.println("============ \n ref in queue");
}

```

打印结果为：

> ============ ref in queue

#### 基础知识——hprof文件

hprof 文件可以展示某一时刻java堆的使用情况，根据这个文件我们可以分析出哪些对象占用大量内存和未在合适时机释放，从而定位内存泄漏问题。

Android 生成 hprof 文件整体上有两种方式:

1. 使用 adb 命令

```
adb shell am dumpheap <processname> <FileName>
复制代码
```

1. 使用 android.os.Debug.dumpHprofData 方法 直接使用 Debug 类提供的 dumpHprofData 方法即可。

```
Debug.dumpHprofData(heapDumpFile.getAbsolutePath());

```

Android Studio 自带 Android Profiler 的 Memory 模块的 dump 操作使用的是方法一。这两种方法生成的 .hprof 文件都是 Dalvik 格式，需要使用 AndroidSDK 提供的 hprof-conv 工具转换成J2SE HPROF格式才能在MAT等标准 hprof 工具中查看。

```
hprof-conv dump.hprof converted-dump.hprof  

```

至于hprof内部格式如何，本文不做具体介绍，以后有机会再单独写一篇文章来仔细讲解。LeakCanary 解析 .hprof 文件用的是 square 公司开源的另一项目：[haha](https://github.com/square/haha).

#### watch方法

终于到了 LeakCanary 关键部分了。我们从 watch 方法入手，前面的代码都是为了增强鲁棒性，我们直接从生成唯一id开始，LeakCanary 构造了一个带有 key 的弱引用对象，并且将 queue 设置为弱引用对象的引用队列。

> 这里解释一下，为什么需要创建一个带有 key 的弱引用对象，不能直接使用 WeakReference 么？ 举个例子，假设 OneActivity 发生了内存泄漏，那么执行 GC 操作时，肯定不会回收 Activity 对象，这样 WeakReference 对象也不会被回收。假设当前启动了 N 个 OneActivity，Dump内存时我们可以获取到内存中的所有 OneActivity，但是当我们准备去检测其中某一个 Activity 的泄漏问题时，我们就无法匹配。但是如果使用了带有 key 的 WeakReference 对象，发生泄露时泄漏时，key 的值也会 dump 保存下来，这样我们根据 key 的一一对应关系就能映射到某一个 Activity。

然后，LeakCanary 调用了 ensureGoneAsync 方法去检测内存泄漏。

```java
public void watch(Object watchedReference, String referenceName) {
  if (this == DISABLED) {
    return;
  }
  checkNotNull(watchedReference, "watchedReference");
  checkNotNull(referenceName, "referenceName");
  final long watchStartNanoTime = System.nanoTime();
  // 对当前监视对象设置一个唯一 id
  String key = UUID.randomUUID().toString();
  // 添加到 Set<String> 中
  retainedKeys.add(key);
  // 构造一个带有id 的 WeakReference 对象
  final KeyedWeakReference reference =
      new KeyedWeakReference(watchedReference, key, referenceName, queue);
  // 检测对象是否被回收了
  ensureGoneAsync(watchStartNanoTime, reference);
}
复制代码
```

#### ensureGoneAsync 方法

ensureGoneAsync 方法构造了一个 Retryable 对象，并将它传给 watchExecutor 的 execute 方法。

```java
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
  // watchExecutor 是 AndroidWatchExecutor的一个实例
  watchExecutor.execute(new Retryable() {
    @Override public Retryable.Result run() {
      return ensureGone(reference, watchStartNanoTime);
    }
  });
}

```

`watchExecutor` 是 AndroidWatchExecutor 的一个实例， AndroidWatchExecutor 的 execute 方法的作用就是判断当前线程是否是主线程，如果是主线程，那么直接执行 waitForIdle 方法，否则通过 Handler 的 post 方法切换到主线程再执行 waitForIdle 方法。

```java
@Override public void execute(@NonNull Retryable retryable) {
  // 判断当前线程是否是主线程
  if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
    waitForIdle(retryable, 0);
  } else {
    postWaitForIdle(retryable, 0);
  }
}

```

`waitForIdle` 方法通过调用 `addIdleHandler` 方法，指定当主进程中没有需要处理的事件时，在这个空闲期间执行 `postToBackgroundWithDelay` 方法。

```java
private void waitForIdle(final Retryable retryable, final int failedAttempts) {
  // 由于上面的 execute 方法，已经保证了此方法在主线程中执行，所以Looper.myQueue()获取的主线程的消息队列
  Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
    @Override public boolean queueIdle() {
      postToBackgroundWithDelay(retryable, failedAttempts);
      // return false 表示执行完之后，就立即移除这个事件
      return false;
    }
  });
}
复制代码
```

`postToBackgroundWithDelay` 方法首先会计算延迟时间 `delayMillis`，这个延时是有 exponentialBackoffFactor（指数因子） 乘以初始延时时间得到的， exponentialBackoffFactor（指数因子）会在2^n 和 Long.MAX_VALUE / initialDelayMillis 中取较小值，也就说延迟时间`delayMillis = initialDelayMillis * 2^n`，且不能超过 Long.MAX_VALUE。

```java
private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
   long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
   // 计算延迟时间
   long delayMillis = initialDelayMillis * exponentialBackoffFactor;
   // 切换到子线程中执行
   backgroundHandler.postDelayed(new Runnable() {
     @Override public void run() {
       // 执行 retryable 里的 run 方法
       Retryable.Result result = retryable.run();
       // 如果需要重试，那么再添加到主线程的空闲期间执行
       if (result == RETRY) {
         postWaitForIdle(retryable, failedAttempts + 1);
       }
     }
   }, delayMillis);
}

```

`postToBackgroundWithDelay` 方法每次执行会指数级增加延时时间，延时时间到了后，会执行 Retryable 里的方法，如果返回为重试，那么会增加延时时间并执行下一次。

`retryable.run()` 的run 方法又执行了什么呢？别忘了我们`ensureGoneAsync`中的代码，一直在重试的代码正式 `ensureGone` 方法。

```java
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
  watchExecutor.execute(new Retryable() {
    @Override public Retryable.Result run() {
      return ensureGone(reference, watchStartNanoTime);
    }
  });
}
```

#### ensureGone 方法

我现在讲`ensureGone`方法的完整代码贴出来，我们逐行分析：

```java
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
  long gcStartNanoTime = System.nanoTime();
  // 前面不是有一个重试的机制么，这里会计下这次重试距离第一次执行花了多长时间
  long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
  // 移除所有弱引用可达对象，后面细讲
  removeWeaklyReachableReferences();
  // 判断当前是否正在开启USB调试，LeakCanary 的解释是调试时可能会触发不正确的内存泄漏
  if (debuggerControl.isDebuggerAttached()) {
    // The debugger can create false leaks.
    return RETRY;
  }
  // 上面执行 removeWeaklyReachableReferences 方法，判断是不是监视对象已经被回收了，如果被回收了，那么说明没有发生内存泄漏，直接结束
  if (gone(reference)) {
    return DONE;
  }
  // 手动触发一次 GC 垃圾回收
  gcTrigger.runGc();
  // 再次移除所有弱引用可达对象
  removeWeaklyReachableReferences();
  // 如果对象没有被回收
  if (!gone(reference)) {
    long startDumpHeap = System.nanoTime();
    long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
    // 使用 Debug 类 dump 当前堆内存中对象使用情况
    File heapDumpFile = heapDumper.dumpHeap();
    // dumpHeap 失败的话，会走重试机制
    if (heapDumpFile == RETRY_LATER) {
      // Could not dump the heap.
      return RETRY;
    }
    long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
    // 将hprof文件、key等属性构造一个 HeapDump 对象
    HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
        .referenceName(reference.name)
        .watchDurationMs(watchDurationMs)
        .gcDurationMs(gcDurationMs)
        .heapDumpDurationMs(heapDumpDurationMs)
        .build();
    // heapdumpListener 分析 heapDump 对象
    heapdumpListener.analyze(heapDump);
  }
  return DONE;
}
复制代码
```

看完上述代码，基本把检测泄漏的大致过程走了一遍，下面我们来看一些具体的细节。

**(1). removeWeaklyReachableReferences 方法**

`removeWeaklyReachableReferences` 移除所有弱引用可达对象是怎么工作的？

```java
private void removeWeaklyReachableReferences() {
  KeyedWeakReference ref;
  while ((ref = (KeyedWeakReference) queue.poll()) != null) {
    retainedKeys.remove(ref.key);
  }
}

```

还记得我们在 refWatcher.watch 方法保存了当前监视对象的 ref.key 了么，如果这个对象被回收了，那么对应的弱引用对象会在回收时被添加到queue中，通过 poll 操作就可以取出这个弱引用，这时候我们从`retainedKeys`中移除这个 key， 代表这个对象已经被正常回收，不需要再被监视了。

那么现在来看，判断这个对象是否被回收就比较简单了？

```java
private boolean gone(KeyedWeakReference reference) {
  // retainedKeys 中不包含 reference.key 的话，就代表这个对象已经被回收了
  return !retainedKeys.contains(reference.key);
}

```

**(2). dumpHeap 方法**

`heapDumper.dumpHeap()` 是执行生成hprof的方法，heapDumper 是 AndroidHeapDumper 的一个对象，我们来具体看看它的 dump 方法。

```java
public File dumpHeap() {
  // 生成一个存储 hprof 的文件
  File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();
  // 文件创建失败
  if (heapDumpFile == RETRY_LATER) {
    return RETRY_LATER;
  }
  // FutureResult 内部有一个 CountDownLatch，用于倒计时
  FutureResult<Toast> waitingForToast = new FutureResult<>();
  // 切换到主线程显示 toast
  showToast(waitingForToast);
  // 等待5秒，确保 toast 已完成显示
  if (!waitingForToast.wait(5, SECONDS)) {
    CanaryLog.d("Did not dump heap, too much time waiting for Toast.");
    return RETRY_LATER;
  }
  // 创建一个通知
  Notification.Builder builder = new Notification.Builder(context)
      .setContentTitle(context.getString(R.string.leak_canary_notification_dumping));
  Notification notification = LeakCanaryInternals.buildNotification(context, builder);
  NotificationManager notificationManager =
      (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
  int notificationId = (int) SystemClock.uptimeMillis();
  notificationManager.notify(notificationId, notification);

  Toast toast = waitingForToast.get();
  try {
    // 开始 dump 内存到指定文件
    Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
    cancelToast(toast);
    notificationManager.cancel(notificationId);
    return heapDumpFile;
  } catch (Exception e) {
    CanaryLog.d(e, "Could not dump heap");
    // Abort heap dump
    return RETRY_LATER;
  }
}

```

这段代码里我们需要看看 `showToast()` 方法，以及它是如何确保 toast 已完成显示（有点黑科技的感觉）。

```java
private void showToast(final FutureResult<Toast> waitingForToast) {
  mainHandler.post(new Runnable() {
    @Override public void run() {
      // 当前 Activity 已经 paused的话，直接返回
      if (resumedActivity == null) {
        waitingForToast.set(null);
        return;
      }
      // 构建一个toast 对象
      final Toast toast = new Toast(resumedActivity);
      toast.setGravity(Gravity.CENTER_VERTICAL, 0, 0);
      toast.setDuration(Toast.LENGTH_LONG);
      LayoutInflater inflater = LayoutInflater.from(resumedActivity);
      toast.setView(inflater.inflate(R.layout.leak_canary_heap_dump_toast, null));
      // 将toast加入显示队列
      toast.show();
      // Waiting for Idle to make sure Toast gets rendered.
      // 主线程中添加空闲时操作，如果主线程是空闲的，会将CountDownLatch执行 countDown 操作
      Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
        @Override public boolean queueIdle() {
          waitingForToast.set(toast);
          return false;
        }
      });
    }
  });
}
```

首先我们需要知道所有的Toast对象，并不是我们一调用 show 方法就立即显示的。`NotificationServiceManager`会从`mToastQueue`中轮询去除Toast对象进行显示。如果Toast的显示不是实时的，那么我们如何知道Toast是否已经显示完成了呢？我们在 Toast 调用 show 方法后调用 addIdleHandler， 在主进程空闲时执行 CountDownLatch 的减一操作。由于我们知道我们顺序加入到主线程的消息队列中的操作：先显示Toast，再执行 CountDownLatch 减一操作。所以在 `if (!waitingForToast.wait(5, SECONDS))` 的判断中，我们最多等待5秒，如果超时会走重试机制，如果我们的 CountDownLatch 已经执行了减一操作，则会正常走后续流程，同时我们也能推理出它前面 toast 肯定已经显示完成了。



`Debug.dumpHprofData(heapDumpFile.getAbsolutePath());`是系统Debug类提供的方法，我就不做具体分析了。



**(3). heapdumpListener.analyze(heapDump) 方法**

heapdumpListener 是 ServiceHeapDumpListener 的一个对象，最终执行了`HeapAnalyzerService.runAnalysis`方法。

```java
@Override public void analyze(@NonNull HeapDump heapDump) {
  checkNotNull(heapDump, "heapDump");
  HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
}
复制代码
// 启动前台服务
public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
  setEnabledBlocking(context, HeapAnalyzerService.class, true);
  setEnabledBlocking(context, listenerServiceClass, true);
  Intent intent = new Intent(context, HeapAnalyzerService.class);
  intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
  intent.putExtra(HEAPDUMP_EXTRA, heapDump);
  ContextCompat.startForegroundService(context, intent);
}
复制代码
```

HeapAnalyzerService 继承自 IntentService，IntentService的具体原理我就不多做解释了。IntentService会将所有并发的启动服务操作，变成顺序执行 onHandleIntent 方法。

```java
@Override protected void onHandleIntentInForeground(@Nullable Intent intent) {
  if (intent == null) {
    CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
    return;
  }
  // 监听 hprof 文件分析结果的类
  String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
  // hprof 文件类
  HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

  HeapAnalyzer heapAnalyzer =
      new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);
  // checkForLeak 会调用 haha 组件中的工具，分析 hprof 文件
  AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
      heapDump.computeRetainedHeapSize);
  // 将分析结果发送给监听器 listenerClassName
  AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
}
复制代码
```

我们来看下 `checkForLeak` 方法，我们一起来看下吧。

```java
public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile,
      @NonNull String referenceKey,
      boolean computeRetainedSize) {
  long analysisStartNanoTime = System.nanoTime();
  // 文件不存在的话，直接返回
  if (!heapDumpFile.exists()) {
    Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
    return failure(exception, since(analysisStartNanoTime));
  }

  try {
    // 更新进度回调
    listener.onProgressUpdate(READING_HEAP_DUMP_FILE);
    // 将 hprof 文件解析成 Snapshot
    HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
    HprofParser parser = new HprofParser(buffer);
    listener.onProgressUpdate(PARSING_HEAP_DUMP);
    Snapshot snapshot = parser.parse();
    listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
    // 移除相同 GC root项
    deduplicateGcRoots(snapshot);
    listener.onProgressUpdate(FINDING_LEAKING_REF);
    // 查找内存泄漏项
    Instance leakingRef = findLeakingReference(referenceKey, snapshot);

    // False alarm, weak reference was cleared in between key check and heap dump.
    // 没有找到，就代表没有泄漏
    if (leakingRef == null) {
      return noLeak(since(analysisStartNanoTime));
    }
    // 找到泄漏处的引用关系链
    return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
  } catch (Throwable e) {
    return failure(e, since(analysisStartNanoTime));
  }
}
复制代码
```

hprof 文件的解析是由开源项目 haha 完成的，我这里不做过多展开。

`findLeakingReference` 方法是查找泄漏的引用处，我们看下代码：

```java
private Instance findLeakingReference(String key, Snapshot snapshot) {
  // 从 hprof 文件保存的对象中找到所有 KeyedWeakReference 的实例
  ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName());
  if (refClass == null) {
    throw new IllegalStateException(
        "Could not find the " + KeyedWeakReference.class.getName() + " class in the heap dump.");
  }
  List<String> keysFound = new ArrayList<>();
  // 对 KeyedWeakReference 实例列表进行遍历
  for (Instance instance : refClass.getInstancesList()) {
    // 获取每个实例里的所有字段
    List<ClassInstance.FieldValue> values = classInstanceValues(instance);
    // 找到 key 字段对应的值
    Object keyFieldValue = fieldValue(values, "key");
    if (keyFieldValue == null) {
      keysFound.add(null);
      continue;
    }
    // 将 keyFieldValue 转为 String 对象
    String keyCandidate = asString(keyFieldValue);
    // 如果这个对象的 key 和 我们查找的 key 相同，那么返回这个弱对象持有的原对象
    if (keyCandidate.equals(key)) {
      return fieldValue(values, "referent");
    }
    keysFound.add(keyCandidate);
  }
  throw new IllegalStateException(
      "Could not find weak reference with key " + key + " in " + keysFound);
}
复制代码
```

到现在为止，我们已经把 LeakCanary 检测内存泄漏的全部过程的源码看完了。个人认为 LeakCanary 源码写的不错，可读性很高，查找调用关系也比较方便（这里黑一下 bilibili 的 DanmakusFlameMaster）。




## 注意点

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


### LeakCanray 2.0为啥不需要在application里调install？

设计ContentProvider的启动流程

初始化是通过ContentProvider来实现，涉及ContentProvider

https://juejin.cn/post/6844903876043210759#heading-11

https://mp.weixin.qq.com/s?__biz=MzUzODQxMzYxNQ==&mid=2247483754&idx=1&sn=f5ea640f40d0158b88a0971bca63bdab&chksm=fad95e2acdaed73c05b4dd8a815666faa5a7b7dba3b9d54b615f52815fb14fb1c976ab6d740d&cur_album_id=1431352953037979650&scene=189#rd



![image-20210416101336383](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210416101336.png)



# 参考

https://www.bilibili.com/video/BV1kK4y18767?p=1

https://search.bilibili.com/all?keyword=LeakCanary&from_source=webtop_search&spm_id_from=333.851


# java 源码系列 - 带你读懂 Reference 和 ReferenceQueue

https://sivanliu.github.io/2017/12/16/provider%E5%88%9D%E5%A7%8B%E5%8C%96/



https://mp.weixin.qq.com/s?__biz=MzUzODQxMzYxNQ==&mid=2247483754&idx=1&sn=f5ea640f40d0158b88a0971bca63bdab&chksm=fad95e2acdaed73c05b4dd8a815666faa5a7b7dba3b9d54b615f52815fb14fb1c976ab6d740d&cur_album_id=1431352953037979650&scene=189#rd





https://mp.weixin.qq.com/s?__biz=MzUzODQxMzYxNQ==&mid=2247483753&idx=1&sn=4f09601c8dab18011373e067a914741c&chksm=fad95e29cdaed73fdd95b69f8a82ce154f9e289b014996ca39b58504e9f3a8fb99355b6614e1&scene=178&cur_album_id=1431352953037979650#rd





# LeakCanray 2.0为啥不需要在application里调install？（B站）



# 反思

之所以理解有问题

1、对流程理解错误   （第一次未整理出正确的流程，导致第二次的时候回忆的流程是错误的）


# 参考

https://juejin.cn/post/6844903730190483470#heading-13




# 享学-如何满足对稳定性要求极高的系统应用开发
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303152035221.png)

## 使用
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303152036408.png)


