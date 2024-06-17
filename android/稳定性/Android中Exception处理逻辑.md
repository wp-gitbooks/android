从整个Android系统来讲，发生的问题主要有3种：

- Exception问题
- ANR问题
- WatchDog问题

本文介绍的主要内容是：

- Android Exception分类
- Exception监听器初始化
- Exception监听器回调流程
- Android Crash弹窗流程

# 一、Exception问题

Exception问题细分的话，也可以可以分成：

- JE：java Exception问题
- NE：native Exception问题
- KE：kernel Exception问题

从问题的深入程度而言，kernel exception已经深入到linux kernel底层，并且和芯片层有交互，这一块直接和驱动交互，问题比较复杂，不像JE和NE可以直接抓一些Exception log，KE的log也能抓上来一些，但是不全，解决的问题依赖的方面太多：kernel层、芯片层、驱动层等等，考虑的因素比较多，不像软件层面的单一化。 本文我们不过多讨论KE的问题，从Exception的抓取调用是如何实现的来分析一下Android源码这方面是如何做的。

## 1.1 思维发散

这里提一个问题，如果让你实现一个监听Exception手机功能，你会有什么思路？

我想大多数人应该知道这样的道理：我可以在系统刚刚启动的时候，设置一个监听器啊，监听所有的进程，一旦某一个进程发生了Exception问题，这个监听器就能监听到，然后触发一个回调调上来，上层就会知道发生了什么问题，然后进一步对这个问题进行处理。 这个是一个大概的设想，而系统Crash的收集模型基本上是按照这样的设计方式来实现的。接下来我们通过分析源码来搞清楚我们是如何收集Exception问题的。


## 1.2 Exception接收器初始化

既然是在系统启动的时候，那么肯定在刚开始的时候就启动了这个收集器了，我们从分析Android系统启动的流程着手。 **1.2.1 系统启动** 我们知道Android系统是init进程启动的，通过解析init.zygote64.rc(看你机器是32位还是64位的)中的信息，启动一个zygote进程。

```java
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server 
```

解析这一行字段，将此参数传入app_main.cpp的main函数中： 看一下/frameworks/base/cmds/app_process/app_main.cpp，main函数中执行了什么？

```java
int main(int argc, char* const argv[])
{
    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
//......
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }
//......
    }
//......
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

代码比较长，我们看重点，就是最后几行：

```java
if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } 
```

这时的zygote显然是true，执行的这句话会调用到ZygoteInit.java中main函数中，这儿也通常被认为是android系统的入口，zygote进程也被认为是android的第一个进程。 **1.2.2 Exception注册器启动** 从启动zygote进程来讲解一下这个监听器的是如何被初始化的。 下面是Exception监听器的初始化过程：

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210809111638.png)

下面从几个启动过程中的关键点来分析一下：

- RuntimeInit.enableDdms(); zygote初始化的时候，开始就会执行这个操作，DDMS全称是Dalvik Debug Monitor Service，是Android开发环境中Dalvik虚拟机的调试监控服务，这个函数是启动DDMS，主要的执行执行代码如下：

```javascript
static final void enableDdms() {
        // Register handlers for DDM messages.
        android.ddm.DdmRegister.registerHandlers();
    }

public static void registerHandlers() {
        if (false)
            Log.v("ddm", "Registering DDM message handlers");
        DdmHandleHello.register();
        DdmHandleThread.register();
        DdmHandleHeap.register();
        DdmHandleNativeHeap.register();
        DdmHandleProfiling.register();
        DdmHandleExit.register();
        DdmHandleViewDebug.register();

        DdmServer.registrationComplete();
    }

public static void registrationComplete() {
        // sync on mHandlerMap because it's convenient and makes a kind of
        // sense
        synchronized (mHandlerMap) {
            mRegistrationComplete = true;
            mHandlerMap.notifyAll();
        }
    }
```

这里的主要目的是注册所有已知的Java VM的处理块的监听器。线程监听、内存监听、native 堆内存监听、debug模式监听等等。

- handleSystemServerProcess 当启动system_server进程成功的时候，需要初始化crash的监听器了。
- RuntimeInit.commonInit() 这个函数是初始化crash监听器，核心的代码如下。

```java
LoggingHandler loggingHandler = new LoggingHandler();
RuntimeHooks.setUncaughtExceptionPreHandler(loggingHandler);
Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));
```

这里有两个接口回调：设置的handlers，对Java 虚拟机的所有线程都起作用，应用程序只能使用Thread.setDefaultUncaughtExceptionHandler，这也是一般的问题收集器会这样使用的。但是RuntimeHooks.setUncaughtExceptionPreHandler这个必须要修改rom才能使用，因为这是系统运行加载的时候就需要运行的，所以应用程序无法更改。

## 1.3 Exception监听器

### 1.3.1 Exception监听器介绍
但是需要注意的是，这里有两个exceptionhandler的实现。分别是**PreHandler**和**Handler**。  
需要注意的是：  
1.setDefaultUncaughtExceptionHandler是公开API。应用可通过调用，自定义。  
2.setUncaughtExceptionPreHandler是hidden API。应用无法调用，不能替换


上面已经明确说了有两个crash收集器：

- RuntimeHooks.setUncaughtExceptionPreHandler
- Thread.setDefaultUncaughtExceptionHandler

设置的这个监听器对象是一个接口：

```java
public interface UncaughtExceptionHandler {
        void uncaughtException(Thread t, Throwable e);
    }
```

最终的回调显然是执行uncaughtException函数。 前一个是系统自己使用的，后一个开发者可以自定义监听器，为什么要设置成两个，我觉得有如下的原因：

- 方便手机rom自己定制，毕竟只有rom才可以更改第一个监听器。
- 设置两道门槛，可以在用户真正收集之前，做一些事情，例如更多crash信息的处理等等。

从这两个设置的监听的函数来看，他们都会设置到Thread.java中，然后将它们赋给Thread.java中的变量，最终触发监听其回调的地方在Thread -> dispatchUncaughtException(…)中。

```java
public final void dispatchUncaughtException(Throwable e) {
        // BEGIN Android-added: uncaughtExceptionPreHandler for use by platform.
        Thread.UncaughtExceptionHandler initialUeh =
                Thread.getUncaughtExceptionPreHandler();
        if (initialUeh != null) {
            try {
                initialUeh.uncaughtException(this, e);
            } catch (RuntimeException | Error ignored) {
                // Throwables thrown by the initial handler are ignored
            }
        }
        // END Android-added: uncaughtExceptionPreHandler for use by platform.
        getUncaughtExceptionHandler().uncaughtException(this, e);
    }
```

上面的代码上下执行了两个回调，这分别代表着上面设置的两个监听器回调。

```javascript
initialUeh.uncaughtException(this, e);
getUncaughtExceptionHandler().uncaughtException(this, e);
```

### 1.3.2 Exception监听器调用
上面介绍了Exception监听器是什么？也介绍了Exception监听器的设置过程。接下里我们需要搞清楚的是Exception监听器是如何工作的。简而言之，就是什么情况下会调用dispatchUncaughtException(…)函数执行回调过程。 下面直接给出dispatchUncaughtException函数的调用过程，然后讲解一下什么情况下会执行这个调用。

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210809111726.png)


在system_server进程启动之后，开始了接下来了注册器回调的过程。 FinalizerWatchdogDaemon继承Daemon，Daemon又实现Runnable，所以执行runInternal就是执行线程，关键点就在这个线程中执行了什么？

```java
private static class FinalizerWatchdogDaemon extends Daemon {
        private static final FinalizerWatchdogDaemon INSTANCE = new FinalizerWatchdogDaemon();

        private boolean needToWork = true;  // Only accessed in synchronized methods.

        FinalizerWatchdogDaemon() {
            super("FinalizerWatchdogDaemon");
        }

        @Override public void runInternal() {
            while (isRunning()) {
                if (!sleepUntilNeeded()) {
                    // We have been interrupted, need to see if this daemon has been stopped.
                    continue;
                }
                final Object finalizing = waitForFinalization();
                if (finalizing != null && !VMRuntime.getRuntime().isDebuggerActive()) {
                    finalizerTimedOut(finalizing);
                    break;
                }
            }
        }
}
```

如果当前的终结器被卡在一个实例上，那我们认为发生了异常情况，这时候执行的函数如下：

```javascript
private static void finalizerTimedOut(Object object) {
            // The current object has exceeded the finalization deadline; abort!
            String message = object.getClass().getName() + ".finalize() timed out after "
                    + (MAX_FINALIZE_NANOS / NANOS_PER_SECOND) + " seconds";
            Exception syntheticException = new TimeoutException(message);
            // We use the stack from where finalize() was running to show where it was stuck.
            syntheticException.setStackTrace(FinalizerDaemon.INSTANCE.getStackTrace());

            // Send SIGQUIT to get native stack traces.
            try {
                Os.kill(Os.getpid(), OsConstants.SIGQUIT);
                // Sleep a few seconds to let the stack traces print.
                Thread.sleep(5000);
            } catch (Exception e) {
                System.logE("failed to send SIGQUIT", e);
            } catch (OutOfMemoryError ignored) {
                // May occur while trying to allocate the exception.
            }
            if (Thread.getUncaughtExceptionPreHandler() == null &&
                    Thread.getDefaultUncaughtExceptionHandler() == null) {
                System.logE(message, syntheticException);
                System.exit(2);
            }
            Thread.currentThread().dispatchUncaughtException(syntheticException);
        }
    }
```

首先收集一下native的堆栈 然后再次check一下本地的Exception回调是否存在，如果不存在，直接退出 如果存在，那么直接执行Exception回调

## 1.4 Exception处理

### 1.4.1 Exception最终的走向

到此为止，讲清楚了本地的Exception是如何初始化，如何设置回调，如何执行回调的。Exception执行的最终结果是什么？这儿收个尾，解析一下在Exception callback成功回调的情况下执行了什么？ 我们就用系统默认情况下采用的KillAplicationHandler来解析一下。其中的uncaughtException函数执行如下：

```java
public void uncaughtException(Thread t, Throwable e) {
            try {
                ensureLogging(t, e);
                if (mCrashing) return;
                mCrashing = true;
                if (ActivityThread.currentActivityThread() != null) {
                    ActivityThread.currentActivityThread().stopProfiling();
                }
                ActivityManager.getService().handleApplicationCrash(
                        mApplicationObject, new ApplicationErrorReport.ParcelableCrashInfo(e));
            } catch (Throwable t2) {
                if (t2 instanceof DeadObjectException) {
                    // System process is dead; ignore
                } else {
                    try {
                        Clog_e(TAG, "Error reporting crash", t2);
                    } catch (Throwable t3) {
                        // Even Clog_e() fails!  Oh well.
                    }
                }
            } finally {
                // Try everything to make sure this process goes away.
                Process.killProcess(Process.myPid());
                System.exit(10);
            }
        }
```

当前线程如果存在的情况下，那么执行 ActivityThread.currentActivityThread().stopProfiling()，此函数是立即停止线程的追踪工作。因为当前线程要发生问题了。 Binder通信到AMS，开始到system_server进程中去重置一些系统变量或者其他方面的操作。

### 1.4.2 Exception处理

Exception的处理流程到AMS中，会经历那些过程，下面还是通过一些序列图来解析一下Exception处理中每一步做了什么事情。

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210809111811.png)

- findAppProcess 根据当前传入的binder信息寻找当前出现问题的ProcessRecord信息，获取当前的ProcessRecord对象。
- addErrorToDropBox 将上面获取的ProcessRecord对象传入，写入发生的问题的堆栈信息和其他的异常相关的信息，保存起来。
- AppErrors.crashApplicationInner 这里对当前的crash情况做了一定的区分。 1.如果当前的进程不存在，那么直接return，不作任何处理。 2.如果当前的进程存在，那么需要弹出Crash dialog，进入下一步
- 发送SHOW_ERROR_UI_MSG的message到AMS中，AMS中定义一个mUiHandler对象，用来分发UI相关的message，这儿即将弹出Crash Dialog，所以需要通过mUiHandler分发。
- AppErrors.handleShowAppErrorUi 最终弹出crash dialog的地方，前面已经获取了crash相关的信息，并且也判断了异常的情况，接下来直接弹出异常的dialog，一般手机的rom系统都是自定义了自己特有的dialog


# 参考

https://cloud.tencent.com/developer/article/1379996