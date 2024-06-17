---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 线索

- Android中 crash 初始化流程图
- crash 监控

# 2 crash概述
[[Android中Exception处理逻辑]]
## 2.1 crash调用链

```java
AMP.handleApplicationCrash
    AMS.handleApplicationCrash
        AMS.findAppProcess
        AMS.handleApplicationCrashInner
            AMS.addErrorToDropBox
            AMS.crashApplication
                AMS.makeAppCrashingLocked
                    AMS.startAppProblemLocked
                    ProcessRecord.stopFreezingAllLocked
                        ActivityRecord.stopFreezingScreenLocked
                            WMS.stopFreezingScreenLocked
                                WMS.stopFreezingDisplayLocked
                    AMS.handleAppCrashLocked
                mUiHandler.sendMessage(SHOW_ERROR_MSG)

Process.killProcess(Process.myPid());
System.exit(10);
```




## 2.2 crash处理流程图
[[Android中Exception处理逻辑#1 2 Exception接收器初始化]]
### 2.2.1 起点RuntimeInit.commonInit

```plantuml
ZygoteInit -> ZygoteInit:main
ZygoteInit -> ZygoteInit:startSystemServer
ZygoteInit -> ZygoteInit:handleSystemServerProcess
ZygoteInit -> RuntimeInit:zygoteInit
RuntimeInit -> RuntimeInit:commonInit()
RuntimeInit -> Thread:setDefaultUncaughtExceptionHandler
```


```java
public class RuntimeInit {
    ...
    private static final void commonInit() {
        //设置默认的未捕获异常处理器，UncaughtHandler实例化过程【见小节2】
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());
        ...
    }
}
```

当进程抛出未捕获异常时，则系统会处理该异常并进入crash处理流程。

![app_crash](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210427155609.jpg)

其中最为核心的工作图中红色部分`AMS.handleAppCrashLocked`的主要功能：

1. 当同一进程1分钟之内连续两次crash，则执行的情况下：
   - 对于非persistent进程：
     - ASS.handleAppCrashLocked, 直接结束该应用所有activity
     - AMS.removeProcessLocked，杀死该进程以及同一个进程组下的所有进
     - ASS.resumeTopActivitiesLocked，恢复栈顶第一个非finishing状态的activity
   - 对于persistent进程，则只执行
     - ASS.resumeTopActivitiesLocked，恢复栈顶第一个非finishing状态的activity
2. 否则，当进程没连续频繁crash
   - ASS.finishTopRunningActivityLocked，执行结束栈顶正在运行activity

另外，`AMS.handleAppCrashLocked`，该方法内部主要调用链，如下：

```java
AMS.handleAppCrashLocked
   ASS.handleAppCrashLocked
       AS.handleAppCrashLocked
           AS.finishCurrentActivityLocked
   AMS.removeProcessLocked
       ProcessRecord.kill
       AMS.handleAppDiedLocked
           ASS.handleAppDiedLocked
               AMS.cleanUpApplicationRecordLocked
               AS.handleAppDiedLocked
                   AS.removeHistoryRecordsForAppLocked

   ASS.resumeTopActivitiesLocked
       AS.resumeTopActivityLocked
           AS.resumeTopActivityInnerLocked
   ASS.finishTopRunningActivityLocked
       AS.finishTopRunningActivityLocked
           AS.finishActivityLocked
```



## 2.3 Binder死亡通知

当crash进程执行kill操作后，进程被杀。此时需要掌握binder 死亡通知原理，由于Crash进程中拥有一个Binder服务端`ApplicationThread`，而应用进程在创建过程调用attachApplicationLocked()，从而attach到system_server进程，在system_server进程内有一个`ApplicationThreadProxy`，这是相对应的Binder客户端。当Binder服务端`ApplicationThread`所在进程(即Crash进程)挂掉后，则Binder客户端能收到相应的死亡通知，从而进入binderDied流程。更多关于bInder原理，这里就不细说，博客中有关于binder系列的专题。

![binder_died](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210427155643.jpg)

# 3 应用

## 3.1 Android中添加全局的Crash监控实战

### 3.1.1 创建一个自定义UncaughtExceptionHandler类

```
public class CrashHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread thread, Throwable ex) {
        //回调函数，处理异常
        //在这里将崩溃日志读取出来，然后保存到SD卡，或者直接上传到日志服务器
        
        //如果我们也想继续调用系统的默认处理，可以先把系统UncaughtExceptionHandler存下来，然后在这里调用。
    }
}
```

### 3.1.2 设置全局监控

```
CrashHandler crashHandler = new CrashHandler();
Thread.setDefaultUncaughtExceptionHandler(crashHandler);
```

完成以上2个步骤，我们就可以实现全局的Crash监控了。这里所说的全局，是指针对整个进程生效。


## 3.2 屏蔽crash提示框

### 3.2.1 源码Framework阶段修改



### 3.2.2 自定义UncaughtExceptionHandler类


## 3.3 如何保证程序不崩溃

### 3.3.1 主线程

可以通过以下代码catch住主线程的异常，代码可以放在生命周期的方法中

```
new Handler((Looper.getMainLooper())).post(new Runnable() {
    @Override
    public void run() {
            for (;;) {
                try {
                Looper.loop();
                } catch (Throwable e) {
                    Log.d("jackie", "==================");
                    e.printStackTrace();
                }
            }

    }
});
```

原理就是通过Handler往主线程的MessageQueue中添加一个Runnable，当主线程执行到该Runnable时，就会进入我们的死循环，如果循环中是空的就会导致代码卡在这里，最终导致ANR，但是我们在while死循环中有调用了Looper.loop()，这就导致主循环又开始不断拿的读取queue中的Message并执行，这样就可以保证以后主线程的所有异常都会从我们手动调用的Looper.loop()中抛出，一旦抛出就会被catch住，这样主线程就不会crash了。

为什么要通过new Handler.post方式而不是直接在主线程中任意位置执行 while (true) { try { Looper.loop(); } catch (Throwable e) {} }。原因是该方法是个死循环，如果在onCreate中执行会导致while后面的代码得不到执行，通过Handler.post方式可以保证不影响该条消息中后面的逻辑。其实感觉用MessageQueue.IdleHandler也不错，在队列空闲时做这个。


### 3.3.2 子线程

```java
//所有的线程异常拦截，由于主线程的异常都被我们catch住了，所有下面的代码拦截到的都是子线程的异常
Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler(){

    @Override
    public void uncaughtException(@NonNull Thread t, @NonNull Throwable e) {
        //处理异常
        Log.d("jackie","============"+Thread.currentThread().getName());
    }
});
```

因为之前已经catch住主线程的异常了，所以上面的代码拦截到的都是子线程的异常了，在uncaughtException方法中我们可以进行上报异常，然后也可以**选择处理异常的策略**，比如该异常足够严重，我们可以直接退出程序，或者让程序继续运行，上面的主线程也可以这样

### 3.3.3 拦截生命周期的异常

简单来说是替换了ActivityThread.mH.mCallback，当然不同的Android版本需要进行适配

Activity 生命周期所有方法都是在`mH`的`handleMessage`方法中调用的，只要能拦截这个`handleMessage`方法就能拦截所有生命周期的异常。然而我们没法通过反射替换掉这个`mH`对象。因为`mH`是 ActivityThread 中一个 H 类的实例，H 类又继承自`Handler`，H 类又是 ActivityThread 中的一个私有类，但是`Handler`会在调用`handleMessage`前调用`mCallback.handleMessage`，`mCallback`是可以被替换掉的

```java
Class activityThreadClass = Class.forName("android.app.ActivityThread");
        Object activityThread = activityThreadClass.getDeclaredMethod("currentActivityThread").invoke(null);

        Field mhField = activityThreadClass.getDeclaredField("mH");
        mhField.setAccessible(true);
        final Handler mhHandler = (Handler) mhField.get(activityThread);
        Field callbackField = Handler.class.getDeclaredField("mCallback");
        callbackField.setAccessible(true);
//mhHandler是ActivityThread.mH，callbackField 是 mH 中的 mCallback 字段，可以通过反射得到
        callbackField.set(mhHandler, new Handler.Callback() {
            //拦截到生命周期相关的消息
            @Override
            public boolean handleMessage(Message msg) {
                switch (msg.what) {
                    case LAUNCH_ACTIVITY:
                        try {
                            //调用ActivityThread.mH.handleMessage
                            mhHandler.handleMessage(msg);
                            return true;
                        } catch (Throwable throwable) {
                            //捕获到生命周期的异常，可以直接关闭该Activity，参考下文的 finish Activity生命周期异常的Activity
                        }
                        //...省略部分相似逻辑
                }
                return false;
            }
        });
```


# 4 崩溃优化
[[崩溃优化]]