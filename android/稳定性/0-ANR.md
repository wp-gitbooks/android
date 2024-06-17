# 线索
如何进行线索ANR监控？

![image-20210429111425141](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210506183137.png)

# 概述

## 定义

**ANR(Application Not responding)，是指应用程序未响应**，Android系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成ANR。一般地，这时往往会弹出一个提示框，告知用户当前xxx未响应，用户可选择继续等待或者Force Close。

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210429094000.png)



## 分类（ANR的产生原因）

从分析源码中得到的这些分类。

|         ANR类型          |              超时时间               |               报错信息               |
| :----------------------: | :---------------------------------: | :----------------------------------: |
| 输入事件（按键、触摸等） |                 5s                  |  Input event dispatching timed out   |
|  广播BroadcastReceiver   |      前台10s，后台/offload 60s      |      Receiver during timeout of      |
|       Service服务        | Foreground 10s，普通 20s，后台 200s |      Timeout executing service       |
|     ContentProvider      |                 10s                 | timeout publishing content providers |

出现ANR的一般有以下几种类型：  
1:**KeyDispatchTimeout**（常见）  
input事件在`5S`内没有处理完成发生了ANR。  
logcat日志关键字：`Input event dispatching timed out`  

2:**BroadcastTimeout**  
前台Broadcast：onReceiver在`10S`内没有处理完成发生ANR。  
后台Broadcast：onReceiver在`60s`内没有处理完成发生ANR。  
logcat日志关键字：`Timeout of broadcast BroadcastRecord`  

3:**ServiceTimeout**  
前台Service：`onCreate`，`onStart`，`onBind`等生命周期在`20s`内没有处理完成发生ANR。  
后台Service：`onCreate`，`onStart`，`onBind`等生命周期在`200s`内没有处理完成发生ANR  
logcat日志关键字：`Timeout executing service`  

4：**ContentProviderTimeout**  
ContentProvider 在`10S`内没有处理完成发生ANR。 logcat日志关键字：timeout publishing content providers


## 例子（现象）

### 简单

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210429094305.png)

### 复杂

那么哪些场景会造成ANR呢？

- Service Timeout:比如前台服务在20s内未执行完成；
- BroadcastQueue Timeout：比如前台广播在10s内未执行完成
- ContentProvider Timeout：内容提供者,在publish过超时10s;
- InputDispatching Timeout: 输入事件分发超时5s，包括按键和触摸事件。

触发ANR的过程可分为三个步骤: 埋炸弹, 拆炸弹, 引爆炸弹


# 原理

http://gityuan.com/2017/01/01/input-anr/

http://gityuan.com/2016/07/02/android-anr/

http://gityuan.com/2016/12/02/app-not-response/


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210427173256.png)



该方法的主要工作发送delay消息(`SERVICE_TIMEOUT_MSG`). 炸弹已埋下, 我们并不希望炸弹被引爆, 那么就需要在炸弹爆炸之前拆除炸弹.



# 典型ANR(常见ANR) 

## 产生ANR情况

- UI线程(主线程)阻塞
- I/O阻塞
- 网络阻塞；
- onReceiver执行时间超过10s;
- 多线程死锁或主线程的等待


1:主线程频繁进行耗时的IO操作：如数据库读写  
2:多线程操作的死锁，主线程被block；  
3:主线程被Binder 对端block；  
4:`System Server`中WatchDog出现ANR；  
5:`service binder`的连接达到上线无法和和System Server通信  
6:系统资源已耗尽（管道、CPU、IO）

  

## 避免ANR

### 避免主线程阻塞

- UI线程尽量只做跟UI相关的工作
- 尽量用Handler来处理UI线程和其他线程之间的交互;
- 耗时的工作()比如数据库操作，I/O，网络操作)，采用单独的工作线程处理
- 用Handler来处理UIthread和工作thread的交互

UI线程，例如：

- Activity:onCreate(), onResume(), onDestroy(), onKeyDown(), onClick(),etc
- AsyncTask: onPreExecute(), onProgressUpdate(), onPostExecute(), onCancel,etc
- Mainthread handler: handleMessage(), post*(runnable r), etc
- …

ANR分析：需要关注CPU/IO，trace死锁等数据

各大组件生命周期中也应避免耗时操作，如Activity的onCreate()、BroadcastReciever的onRecieve()等。**广播如有耗时操作，建议放到IntentService中执行。**

**注：** IntentService继承了Service，运行在后台线程中，处理异步需求，工作执行完毕后自动销毁。

但Android版本8.0开始限制后台，API level 30被弃用。8.0及更高版本 建议使用WorkManager 或 JobIntentService替代


### 避免CPU负荷或I/O阻塞：

文件或数据操作放到子线程，异步方式完成。



### 各大组件ANR

各大组件生命周期中也应避免耗时操作，注意 Activity、BroadcastReciever、Service 和 ContentProvider 也不要执行耗时的任务。




# 问题分析

## 定位ANR（发现ANR）

### 1进行Monkey测试



### 2 StrictMode

**`StrictMode(严格模式)`** 是 Android SDK 提供的一个用来检测代码中是否存在违规操作的工具类，`StrictMode` 主要检测两大类问题：

- 线程策略：ThreadPolicy：
  - detectNetwork：检测是否存在网络操作
  - detectDiskReads：检测是否存在磁盘读取操作
  - detectDiskWrites：检测是否存在磁盘写入操作
  - detectCustomSlowCalls：检测自定义耗时操作
- 虚拟机策略：VmPolicy
  - detectActivityLeaks：检测是否存在 Activity 泄漏
  - setClassInstanceLimit：检测类实例个数是否超过限制
  - detectLeakedSqliteObjects：检测是否存在 Sqlite 对象泄漏
  - detectLeakedClosableObjects：检测是否存在未关闭的 Closable 对象泄漏

### 3 BlockCanary

**`BlockCanary`** 是一个轻量的、非侵入式的性能监控函数库，它的用法和 `LeakCanary` 类似，只不过后者监控应用的内存泄漏，而 `BlockCanary` 主要用来监控应用主线程的卡顿，并可通过组件提供的各种信息分析出原因并进行修复。它的原理是利用主线程的消息队列处理机制，通过对比消息分发开始和结束的时间点来判断是否超过设定的时间，如果是，则判断为主线程卡顿。


### 4 监控



## 分析（遇到ANR问题如何分析、如何能更快速准确的定位问题？）

### Logcat日志信息

产生 `ANR` 后，立即查看 Log，可以看到 `logcat` 清晰地记录了 `ANR` 发生的时间，以及线程的 `tid` 和一句话概括原因：`WaitingInMainSignalCatcherLoop`，大概意思为主线程等待异常。
最后一句 `The application may be doing too much work on its main thread.`告知可能在主线程做了太多的工作

### events_log
查看mobilelog文件夹下的events_log,从日志中搜索关键字：`am_anr`，找到出现ANR的时间点、进程PID、ANR类型。

如日志：
```apache
07-20 15:36:36.472  1000  1520  1597 I am_anr  : [0,1480,com.xxxx.moblie,952680005,Input dispatching timed out (AppWindowToken{da8f666 token=Token{5501f51 ActivityRecord{15c5c78 u0 com.xxxx.moblie/.ui.MainActivity t3862}}}, Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)]复制代码
```

从上面的log我们可以看出： 应用`com.xxxx.moblie` 在`07-20 15:36:36.472`时间，发生了一次`KeyDispatchTimeout`类型的ANR，它的进程号是`1480`. 把关键的信息整理一下：  
**ANR时间**：07-20 15:36:36.472  
**进程pid**：1480  
**进程名**：com.xxxx.moblie  
**ANR类型**：KeyDispatchTimeout  

我们已经知道了发生`KeyDispatchTimeout`的ANR是因为 `input事件在5秒内没有处理完成`。那么在这个时间`07-20 15:36:36.472` 的前5秒，也就是（`15:36:30 ~15:36:31`）时间段左右程序到底做了什么事情？这个简单，因为我们已经知道pid了，再搜索一下`pid = 1480`的日志.这些日志表示该进程所运行的轨迹，关键的日志如下：

```tap
07-20 15:36:29.749 10102  1480  1737 D moblie-Application: [Thread:17329] receive an intent from server, action=com.ttt.push.RECEIVE_MESSAGE
07-20 15:36:30.136 10102  1480  1737 D moblie-Application: receiving an empty message, drop
07-20 15:36:35.791 10102  1480  1766 I Adreno  : QUALCOMM build                   : 9c9b012, I92eb381bc9
07-20 15:36:35.791 10102  1480  1766 I Adreno  : Build Date                       : 12/31/17
07-20 15:36:35.791 10102  1480  1766 I Adreno  : OpenGL ES Shader Compiler Version: EV031.22.00.01
07-20 15:36:35.791 10102  1480  1766 I Adreno  : Local Branch                     : 
07-20 15:36:35.791 10102  1480  1766 I Adreno  : Remote Branch                    : refs/tags/AU_LINUX_ANDROID_LA.UM.6.4.R1.08.00.00.309.049
07-20 15:36:35.791 10102  1480  1766 I Adreno  : Remote Branch                    : NONE
07-20 15:36:35.791 10102  1480  1766 I Adreno  : Reconstruct Branch               : NOTHING
07-20 15:36:35.826 10102  1480  1766 I vndksupport: sphal namespace is not configured for this process. Loading /vendor/lib64/hw/gralloc.msm8998.so from the current namespace instead.
07-20 15:36:36.682 10102  1480  1480 W ViewRootImpl[MainActivity]: Cancelling event due to no window focus: KeyEvent { action=ACTION_UP, keyCode=KEYCODE_PERIOD, scanCode=0, metaState=0, flags=0x28, repeatCount=0, eventTime=16099429, downTime=16099429, deviceId=-1, source=0x101 }复制代码
```

从上面我们可以知道，在时间 07-20 15:36:29.749 程序收到了一个action消息。

```java
07-20 15:36:29.749 10102  1480  1737 D moblie-Application: [Thread:17329] receive an intent from server, action=com.ttt.push.RECEIVE_MESSAGE。
```

原来是应用`com.xxxx.moblie` 收到了一个推送消息（`com.ttt.push.RECEIVE_MESSAGE`）导致了阻塞，我们再串联一下目前所获取到的信息：在时间`07-20 15:36:29.749` 应用`com.xxxx.moblie` 收到了一下推送信息`action=com.ttt.push.RECEIVE_MESSAGE`发生阻塞，5秒后发生了`KeyDispatchTimeout的ANR`。

虽然知道了是怎么开始的，但是具体原因还没有找到，是不是当时CPU很紧张、各路APP再抢占资源？ 我们再看看CPU的信息,。搜索关键字关键字： `ANR IN`  

```apache
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: ANR in com.xxxx.moblie (com.xxxx.moblie/.ui.MainActivity) (进程名)
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: PID: 1480 (进程pid)
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: Reason: Input dispatching timed out (AppWindowToken{da8f666 token=Token{5501f51 ActivityRecord{15c5c78 u0 com.xxxx.moblie/.ui.MainActivity t3862}}}, Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: Load: 0.0 / 0.0 / 0.0 (Load表明是1分钟,5分钟,15分钟CPU的负载)
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: CPU usage from 20ms to 20286ms later (2018-07-20 15:36:36.170 to 2018-07-20 15:36:56.436):
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   42% 6774/pressure: 41% user + 1.4% kernel / faults: 168 minor
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   34% 142/kswapd0: 0% user + 34% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   31% 1520/system_server: 13% user + 18% kernel / faults: 58724 minor 1585 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   13% 29901/com.ss.android.article.news: 7.7% user + 6% kernel / faults: 56007 minor 2446 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   13% 32638/com.android.quicksearchbox: 9.4% user + 3.8% kernel / faults: 48999 minor 1540 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   11% (CPU的使用率)1480/com.xxxx.moblie: 5.2%(用户态的使用率) user + (内核态的使用率) 6.3% kernel / faults: 76401 minor 2422 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   8.2% 21000/kworker/u16:12: 0% user + 8.2% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   0.8% 724/mtd: 0% user + 0.8% kernel / faults: 1561 minor 9 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   8% 29704/kworker/u16:8: 0% user + 8% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   7.9% 24391/kworker/u16:18: 0% user + 7.9% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   7.1% 30656/kworker/u16:14: 0% user + 7.1% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   7.1% 9998/kworker/u16:4: 0% user + 7.1% kernel复制代码
```

我已经在log 中标志了相关的含义。`com.xxxx.moblie` 占用了11%的CPU，其实这并不算多。现在的手机基本都是多核CPU。假如你的CPU是4核，那么上限是400%，以此类推。

既然不是CPU负载的原因，那么到底是什么原因呢？ 这时就要看我们的终极大杀器——`traces.txt`。


### Traces.txt日志信息

/data/anr/traces.txt

```
----- pid 23346 at 2019-07-08 11:33:57 ----- Cmd line: com.android.anrtest Build fingerprint: 'google/marlin/marlin:8.0.0/OPR3.170623.007/4286350:user/release-keys' ABI: 'arm64' Build type: optimized Zygote loaded classes=4681 post zygote classes=106 Intern table: 42675 strong; 137 weak JNI: CheckJNI is on; globals=526 (plus 22 weak) Libraries: /system/lib64/libandroid.so /system/lib64/libcompiler_rt.so /system/lib64/libjavacrypto.so /system/lib64/libjnigraphics.so /system/lib64/libmedia_jni.so /system/lib64/libsoundpool.so /system/lib64/libwebviewchromium_loader.so libjavacore.so libopenjdk.so (9) Heap: 22% free, 1478KB/1896KB; 21881 objects ... "main" prio=5 tid=1 Sleeping | group="main" sCount=1 dsCount=0 flags=1 obj=0x733d0670 self=0x74a4abea00 | sysTid=23346 nice=-10 cgrp=default sched=0/0 handle=0x74a91ab9b0 | state=S schedstat=( 391462128 82838177 354 ) utm=33 stm=4 core=3 HZ=100 | stack=0x7fe6fac000-0x7fe6fae000 stackSize=8MB | held mutexes= at java.lang.Thread.sleep(Native method) - sleeping on <0x053fd2c2> (a java.lang.Object) at java.lang.Thread.sleep(Thread.java:373) - locked <0x053fd2c2> (a java.lang.Object) at java.lang.Thread.sleep(Thread.java:314) at android.os.SystemClock.sleep(SystemClock.java:122) at com.android.anrtest.ANRTestActivity.onCreate(ANRTestActivity.java:20) at android.app.Activity.performCreate(Activity.java:6975) at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1213) at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2770) at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2892) at android.app.ActivityThread.-wrap11(ActivityThread.java:-1) at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1593) at android.os.Handler.dispatchMessage(Handler.java:105) at android.os.Looper.loop(Looper.java:164) at android.app.ActivityThread.main(ActivityThread.java:6541) at java.lang.reflect.Method.invoke(Native method) at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:240) at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:767)
```


### bugreport


## ANR 案例整理
### 死锁
**一、主线程被其他线程lock，导致死锁**

`waiting on <0x1cd570> (a android.os.MessageQueue)`

```routeros
DALVIK THREADS:
"main" prio=5 tid=3 TIMED_WAIT
  | group="main" sCount=1 dsCount=0 s=0 obj=0x400143a8
  | sysTid=691 nice=0 sched=0/0 handle=-1091117924
  at java.lang.Object.wait(Native Method)
  - waiting on <0x1cd570> (a android.os.MessageQueue)
  at java.lang.Object.wait(Object.java:195)
  at android.os.MessageQueue.next(MessageQueue.java:144)
  at android.os.Looper.loop(Looper.java:110)
  at android.app.ActivityThread.main(ActivityThread.java:3742)
  at java.lang.reflect.Method.invokeNative(Native Method)
  at java.lang.reflect.Method.invoke(Method.java:515)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:739)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:497)
  at dalvik.system.NativeStart.main(Native Method)

"Binder Thread #3" prio=5 tid=15 NATIVE
  | group="main" sCount=1 dsCount=0 s=0 obj=0x434e7758
  | sysTid=734 nice=0 sched=0/0 handle=1733632
  at dalvik.system.NativeStart.run(Native Method)

"Binder Thread #2" prio=5 tid=13 NATIVE
  | group="main" sCount=1 dsCount=0 s=0 obj=0x1cd570
  | sysTid=696 nice=0 sched=0/0 handle=1369840
  at dalvik.system.NativeStart.run(Native Method)

"Binder Thread #1" prio=5 tid=11 NATIVE
  | group="main" sCount=1 dsCount=0 s=0 obj=0x433aca10
  | sysTid=695 nice=0 sched=0/0 handle=1367448
  at dalvik.system.NativeStart.run(Native Method)

----- end 691 -----复制代码
```

### 耗时
**二、主线程做耗时的操作：比如数据库读写。**

```
"main" prio=5 tid=1 Native
held mutexes=
kernel: (couldn't read /proc/self/task/11003/stack)
native: #00 pc 000492a4 /system/lib/libc.so (nanosleep+12)
native: #01 pc 0002dc21 /system/lib/libc.so (usleep+52)
native: #02 pc 00009cab /system/lib/libsqlite.so (???)
native: #03 pc 00011119 /system/lib/libsqlite.so (???)
native: #04 pc 00016455 /system/lib/libsqlite.so (???)
native: #16 pc 0000fa29 /system/lib/libsqlite.so (???)
native: #17 pc 0000fad7 /system/lib/libsqlite.so (sqlite3_prepare16_v2+14)
native: #18 pc 0007f671 /system/lib/libandroid_runtime.so (???)
native: #19 pc 002b4721 /system/framework/arm/boot-framework.oat (Java_android_database_sqlite_SQLiteConnection_nativePrepareStatement__JLjava_lang_String_2+116)
at android.database.sqlite.SQLiteConnection.setWalModeFromConfiguration(SQLiteConnection.java:294)
at android.database.sqlite.SQLiteConnection.open(SQLiteConnection.java:215)
at android.database.sqlite.SQLiteConnection.open(SQLiteConnection.java:193)
at android.database.sqlite.SQLiteConnectionPool.openConnectionLocked(SQLiteConnectionPool.java:463)
at android.database.sqlite.SQLiteConnectionPool.open(SQLiteConnectionPool.java:185)
at android.database.sqlite.SQLiteConnectionPool.open(SQLiteConnectionPool.java:177)
at android.database.sqlite.SQLiteDatabase.openInner(SQLiteDatabase.java:808)
locked <0x0db193bf> (a java.lang.Object)
at android.database.sqlite.SQLiteDatabase.open(SQLiteDatabase.java:793)
at android.database.sqlite.SQLiteDatabase.openDatabase(SQLiteDatabase.java:696)
at android.app.ContextImpl.openOrCreateDatabase(ContextImpl.java:690)
at android.content.ContextWrapper.openOrCreateDatabase(ContextWrapper.java:299)
at android.database.sqlite.SQLiteOpenHelper.getDatabaseLocked(SQLiteOpenHelper.java:223)
at android.database.sqlite.SQLiteOpenHelper.getWritableDatabase(SQLiteOpenHelper.java:163)
locked <0x045a4a8c> (a com.xxxx.video.common.data.DataBaseHelper)
at com.xxxx.video.common.data.DataBaseORM.<init>(DataBaseORM.java:46)
at com.xxxx.video.common.data.DataBaseORM.getInstance(DataBaseORM.java:53)
locked <0x017095d5> (a java.lang.Class<com.xxxx.video.common.data.DataBaseORM>)
```

### binder数据过大
**三、binder数据量过大**

```
07-21 04:43:21.573  1000  1488 12756 E Binder  : Unreasonably large binder reply buffer: on android.content.pm.BaseParceledListSlice$1@770c74f calling 1 size 388568 (data: 1, 32, 7274595)
07-21 04:43:21.573  1000  1488 12756 E Binder  : android.util.Log$TerribleFailure: Unreasonably large binder reply buffer: on android.content.pm.BaseParceledListSlice$1@770c74f calling 1 size 388568 (data: 1, 32, 7274595)
07-21 04:43:21.607  1000  1488  2951 E Binder  : Unreasonably large binder reply buffer: on android.content.pm.BaseParceledListSlice$1@770c74f calling 1 size 211848 (data: 1, 23, 7274595)
07-21 04:43:21.607  1000  1488  2951 E Binder  : android.util.Log$TerribleFailure: Unreasonably large binder reply buffer: on android.content.pm.BaseParceledListSlice$1@770c74f calling 1 size 211848 (data: 1, 23, 7274595)
07-21 04:43:21.662  1000  1488  6258 E Binder  : Unreasonably large binder reply buffer: on android.content.pm.BaseParceledListSlice$1@770c74f calling 1 size 259300 (data: 1, 33, 7274595)
```

### binder通信失败
**四、binder 通信失败**
```
07-21 06:04:35.580 <6>[32837.690321] binder: 1698:2362 transaction failed 29189/-3, size 100-0 line 3042
07-21 06:04:35.594 <6>[32837.704042] binder: 1765:4071 transaction failed 29189/-3, size 76-0 line 3042
07-21 06:04:35.899 <6>[32838.009132] binder: 1765:4067 transaction failed 29189/-3, size 224-8 line 3042
07-21 06:04:36.018 <6>[32838.128903] binder: 1765:2397 transaction failed 29189/-22, size 348-0 line 2916
```


## 解决ANR




# 面试题

## 如何直接定位ANR，堆栈信息都无法直接定位，怎么解决呢(看下系统下的trace)

