---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 参考
http://gityuan.com/android/

# 2 线索
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051916688.png)

# 3 开机执行流程
http://gityuan.com/android/

## 3.1 问题
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051639326.png)
## 3.2 简图
**Bootloader** ➪**Kernel** ➪**Init进程** ➪ **Zygote** ➪ **SystemServer** ➪ **ServiceManager** ➪ **Home Launcher**

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051916688.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051502375.png)

Boot Loader 引导开机，然后依次进入 -> Kernel -> Native -> Framework -> App

## 3.3 详细图

![这里写图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210425174235.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051921044.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051916688.png)

framework基本在用户空间（用户态）

init、zygote 进程在用户态

一般 native(C/C++)、java 都在用户空间(用户态)，而不是内核空间

zygote 既运行在 Java 层、又运行在 native 层

init.rc----->App_main.cpp

zygote native:启动 jvm、art、jvm C/C++



![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051509461.png)





![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210309213347)


![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210309213812)

![img](https://upload-images.jianshu.io/upload_images/2929448-29279bde1a08d782.png?imageMogr2/auto-orient/strip|imageView2/2/w/916/format/webp)

## 3.4 架构层

### 3.4.1 Loader层

- `Boot Rom`：当手机处于关机状态时，长按开机键开机，会引导芯片开始从固化在Rom里预设的代码开始执行，然后加载引导程序到`Ram`
- `Boot Loader`：启动Android系统之前的引导程序，主要是检查`Ram`，初始化参数等

#### 3.4.1.1 BootLoader
板子上电后，芯片从固化在 ROM 里预设的代码(BOOT ROM)开始执行， BOOT ROM 会加载 Bootloader 到 RAM，然后把控制权交给 BootLoader。
BootLoader 并不隶属于 Android 系统，它的作用是初始化硬件设备，加载内核文件等，为 Android 系统内核启动搭建好所需的环境（可以把 Bootloader 类比成 PC 的 BIOS）。
Bootloader 是针对特定的主板与芯片的（与 CPU 及电路板的配置情况有关），因此，对于不同的设备制造商，它们的引导程序都是不同的。目前大多数系统使都是使用 [uboot](http://www.denx.de/wiki/U-Boot)来修改的。
Bootloader 引导程序一般分两个阶段执行：

1. 基本的硬件初始化，目的是为下一阶段的执行以及随后的 kernel 的执行准备好一些基本的硬件环境。这一阶段的代码通常用汇编语言编写，以达到短小精悍的目的。
2. 初始化 Flash 设备，设置网络、内存等等，将 kernel 映像和根文件系统映像从 Flash 上读到 RAM 空间中，然后启动内核。这一阶段的代码通常用 C 语言来实现，以便于实现更复杂的功能和取得更好的代码可读性和可移植性。

实际上 Bootloader 还要根据 misc 分区的设置来决定是要正常启动系统内核还是要进入 recovery 进行系统升级，复位等工作



### 3.4.2 Kernel层

`Kernel`层指的就是Android内核层，这里一般开机刚刚结束进入Android系统，`Kernel`层的启动流程如下：

1. 启动`swapper`进程（pid=0），这是系统初始化过程kernel创建的第一个进程，用于初始化进程管理、内存管理、加载`Display`、`Camera`、`Binder`等驱动相关工作
2. 启动`kthreadd`进程，这是`Linux`系统的内核进程，会创建内核工作线程`kworkder`、软中断线程`ksoftirqd`和`thermal`等内核守护进程。`kthreadd`是所有内核进程的鼻祖。

### 3.4.3 Native层

这里的`native`层主要包括由`init`进程孵化的用户空间的守护进程，`bootanim`开机动画和`hal`层等。`Init`是`Linux`系统的守护进程，是所有用户空间进程的鼻祖。

- `init`进程会孵化出`ueventd`、`logd`、`healthd`、`installd`、`adbd`、lm`这里写代码片`kd等用户守护进程；
- `init`进程还会启动`ServiceManager`（Binder服务管家）、`bootanim`（开机动画）等重要服务。
- `init`进程孵化出`Zygote`进程，`Zygote`进程是Android系统第一个Java进程（虚拟机进程），`Zygote`进程是所有Java进程的父进程。

### 3.4.4 Framework层
`framework`层有`native`层和`java`层共同组成，协调系统平稳有序的工作。`framework` 层主要包括以下内容：

- `Media Server`进程，是由`init`进程`fork`而来，负责启动和管理整个`C++ framework`，包含`AudioFlinger`，`Camera Service`等服务。
- `Zygote`进程，由`Init`进程通过解析`init.rc`文件生成，`Zygote`是Android系统的第一个Java进程，是所有Java进程的父进程。
- `System Server`进程，由`Zygote`进程`fork`而来，是`Zygote`进程孵化的第一个子进程，负责启动和管理整个`Java Framework`，包括`Ams`、`Pms`等。


### 3.4.5 App层
`Zygote`进程孵化的第一个`App`进程是`Launcher`进程，也就是我们的桌面进程，也就是我们打开手机看到的用户界面。因为在前面的`framework` 生成了各种守护进程和管理进程，对于`Launcher`也就有对应的点击、长按、滑动、卸载等监听。`Zygote`进程也会创建`Browser`、`Phone`、`Email`等App进程。也就是说所有的App进程都是由`Zygote`进程`fork`生成的。而且上层的进程全部由下层的进程进行管理，包括但不限于界面的注册、跳转，消息的传递。


# 4 系统启动
http://gityuan.com/android/

![image-20210407190430603](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407190430.png)

## 4.1 进程main方法
| 进程              | 主方法                |
| :---------------- | :-------------------- |
| init进程          | Init.main()           |
| zygote进程        | ZygoteInit.main()     |
| app_process进程   | RuntimeInit.main()    |
| system_server进程 | SystemServer.main()   |
| app进程           | ActivityThread.main() |

# 5 init进程
http://gityuan.com/2016/02/05/android-init/

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303052017453.png)


## 5.1 概述
init进程是Linux系统中用户空间的第一个进程，进程号固定为1。Kernel启动后，在用户空间启动init进程，并调用init中的main()方法执行init进程的职责。对于init进程的功能分为4部分：

- **解析**并运行所有的init.rc相关文件
- 根据rc文件，**生成**相应的设备驱动节点
- 处理子进程的终止(signal方式)
- 提供属性服务的功能

![这里写图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210425174411.png)



## 5.2 启动服务

### 5.2.1 启动顺序

```
on early-init
on init
on late-init
    trigger post-fs      
    trigger load_system_props_action
    trigger post-fs-data  
    trigger load_persist_props_action
    trigger firmware_mounts_complete
    trigger boot   

on post-fs      //挂载文件系统
    start logd
    mount rootfs rootfs / ro remount
    mount rootfs rootfs / shared rec
    mount none /mnt/runtime/default /storage slave bind rec
    ...

on post-fs-data  //挂载data
    start logd
    start vold   //启动vold
    ...

on boot      //启动核心服务
    ...
    class_start core //启动core class
```



### 5.2.2 服务启动(Zygote)
在init.zygote.rc文件中，zygote服务定义如下：

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```


### 5.2.3 服务重启
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303060836866.png)




## 5.3 流程

![zygote_init](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210309093051.jpg)

# 6 Zygote启动流程
http://gityuan.com/2016/02/13/android-zygote/


![zygote_process](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407172747.jpg)



## 6.1 主要工作
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303052014672.png)

调用流程：
```java
App_main.main
    AndroidRuntime.start
        AndroidRuntime.startVm
        AndroidRuntime.startReg
        ZygoteInit.main (首次进入Java世界)
            registerZygoteSocket
            preload
            startSystemServer
            runSelectLoop
```


zygote 的主要工作如下：
- 创建 **java 虚拟机 AndroidRuntime**
- 通过 AndroidRuntime 启动 ZygoteInit 进入 java 环境。

ZygoteInit的主要工作如下：
- 创建 socket 服务，接受 **ActivityManagerService 的应用启动请求**。
- 加载 **Android framework** 中的 class、res（drawable、xml信息、strings）到内存。Android 通过在 zygote 创建的时候加载资源，生成信息链接，再有应用启动，fork 子进程和父进程共享信息，不需要重新加载，同时也共享 VM。
- **启动 SystemServer**。
- 监听 socket，当有启动应用请求到达，fork 生成 App 应用进程。


## 6.2 流程图

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303202117472.png)


1.  解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；
2.  调用AndroidRuntime的startVM()方法**创建虚拟机**，再调用startReg()注册JNI函数；
3.  通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；
4.  registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
5.  preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，用于提高app启动效率；
6.  zygote完毕大部分工作，接下来再通过**startSystemServer()**，fork得力帮手system_server进程，也是上层framework的运行载体。
7.  zygote功成身退，调用**runSelectLoop()**，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作。

![这里写图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210309220708)
## 6.3 详细流程
zygote是什么？

了解zygote进程嘛？他是用了干什么的，如何跟他通信？

zygote是谁启动的？为什么要有zygote进程？（回答：假设没有zygote进程会发生什么？zygote进程的功能或作用）

1、管理
2、为什么需要socket方式而不用binder？
a、安全
b、C/S

LocalSocket

了解了`Android`系统从按下开机键到桌面完整运行在用户眼前的整个流程，我们就可以针对系统的各个过程进行分析。由于是移动开发，平时最多打交道的是应用层，也是就上面的`App`层，跟我们打交道最多的就是`framework`层，我们主要关注`framework`层是如何启动并调度各应用进程协调工作的。从`ZygoteInit`的`main`方法开始，我们先看`framework`启动流程的时序图(省略了一些步骤)大体如下：
![这里写图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210708162915)

### 6.3.1 ZygoteInit.main
```java
 public static void main(String argv[]) {
        ZygoteServer zygoteServer = new ZygoteServer();

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();
		...
        try {
            // Report Zygote start time to tron unless it is a runtime restart
            if (!"1".equals(SystemProperties.get("sys.boot_completed"))) {
                MetricsLogger.histogram(null, "boot_zygote_init",
                        (int) SystemClock.elapsedRealtime());
            }
			...
			// 1.加载首个Zygote进程的时候，加载参数有start-system-server，即startSystemServer=true
			
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }

			// 2. 注册zygote服务端localserversocket
            zygoteServer.registerServerSocket(socketName);
	        ...
			// 3.在这里开启了SystemServer，也就是Ams,Pms,Wms...等一系列系统服务
            if (startSystemServer) {
                startSystemServer(abiList, socketName, zygoteServer);
            }
            Log.i(TAG, "Accepting command socket connections");
            
			// 4.while(true) 死循环，除非抛出异常系统中断
            zygoteServer.runSelectLoop(abiList);
            zygoteServer.closeServerSocket();
        } catch (Zygote.MethodAndArgsCaller caller) {
            caller.run();
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            zygoteServer.closeServerSocket();
            throw ex;
        }
    }

```

`ZygoteInit.main()`由`native`层传入初始化参数调用，这里不再赘述，这边主要沿Java的`framework`层来分析系统的调用，结合上图和代码注释可以看到，加载参数有start-system-server，即startSystemServer=true赋值，紧接着下面会调用`ZygoteServer.registerServerSocket()`,也就是这里和`ZygoteServer`的通信用到的是`LocalSocket`，我们知道跨进程调用最常见的调用方式就是`LocalSocket`和`aidl`，在`framework`里也有很多这样通信的。接下来就开启了`SystemServer`，后面是一个`runSelectLoop()`，这里其实是一个死循环，当抛出异常终止则会调用`zygoteServer.closeServerSocket()`来关闭`socket`连接。我们继续跟进`startSystemServer`看具体是如何开启系统服务的。


### 6.3.2 startSystemServer

```java
    /**
     * Prepare the arguments and fork for the system server process.
     */
    private static boolean startSystemServer(String abiList, String socketName, ZygoteServer zygoteServer)
            throws Zygote.MethodAndArgsCaller, RuntimeException {
        ...
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1032,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
	        //传入准备参数，注意上面参数最后的"com.android.server.SystemServer"
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
		
		// pid = 0 说明创建子进程成功，SystemServer此时拥有独立进程，可以独立在自己的进程内操作了
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            zygoteServer.closeServerSocket();
            // 处理SystemServer进程
            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
复制代码
```

这个方法不是很长，看注释可以知道，主要是用来准备`SystemServer`进程的参数和fork`SystemServer`进程。前面一堆参数不用看，注意我添加中文注释的地方，传入的参数有一个内容为`"com.android.server.SystemServer"`，这是`SystemServer`类的全限定名，接下来调用`Zygote.forkSystemSeerver()`，最后当`pid=0`时，也就是说明子进程创建成功，如果这时候有两个`Zygote`进程则等待第二个`Zygote`进程连接，关闭掉第一个`Zygote`进程和`ZygoteServer`的`socket`连接。最后调用`handleSystemServerProcess()`来处理`SystemServer`进程。这里的`Zygote.forkSystemServer()`实际上是调用了`native`层的`forkSystemServer()`来fork子进程。这里主要跟进`handleSystemServerProcess()`看看是如何完成新fork的`SystemServer`进程的剩余工作的。

### 6.3.3 handleSystemServerProcess

```java
	/**
     * Finish remaining work for the newly forked system server process.
     */
    private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws Zygote.MethodAndArgsCaller {
		...
		// 默认为null，走Zygote.init()流程
		
        if (parsedArgs.invokeWith != null) {
            ...
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
	            // 创建类加载器，并赋予当前线程
                cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);
                Thread.currentThread().setContextClassLoader(cl);
            }

            /*
             * Pass the remaining arguments to SystemServer.
             */
            ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }

        /* should never reach here */
    }

```

`parsedArgs.invokeWith`默认是为null的，也就是走`else`流程，可以看到先创建了一个类加载器并赋予了当前线程，然后进入了`ZygoteInit.zygoteInit()`。

### 6.3.4 RuntimeInit.zygoteInit()

```java
    /**
     * The main function called when started through the zygote process. This
     * could be unified with main(), if the native code in nativeFinishInit()
     * were rationalized with Zygote startup.<p>
     *
     * Current recognized args:
     * <ul>
     *   <li> <code> [--] &lt;start class name&gt;  &lt;args&gt;
     * </ul>
     *
     * @param targetSdkVersion target SDK version
     * @param argv arg strings
     */
    public static final void zygoteInit(int targetSdkVersion, String[] argv,
            ClassLoader classLoader) throws Zygote.MethodAndArgsCaller {
        if (RuntimeInit.DEBUG) {
            Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
        RuntimeInit.redirectLogStreams();

        RuntimeInit.commonInit();
		// Zygote初始化
        ZygoteInit.nativeZygoteInit();
		// 应用初始化
        RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
    }

```

这个方法不是很长，主要工作是`Zygote`的初始化和`Runtime`的初始化。`Zygote` 的初始化调用了`native`方法，里面的大致工作是打开`/dev/binder`驱动设备，创建一个新的`binder`线程，调用`talkWithDriver()`不断地跟驱动交互。 进入`RuntimeInit.applicationInit()`查看具体应用初始化流程。

### 6.3.5 RuntimeInit.applicationInit

```java
    protected static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws Zygote.MethodAndArgsCaller {
        // If the application calls System.exit(), terminate the process
        // immediately without running any shutdown hooks.  It is not possible to
        // shutdown an Android application gracefully.  Among other things, the
        // Android runtime shutdown hooks close the Binder driver, which can cause
        // leftover running threads to crash before the process actually exits.
        nativeSetExitWithoutCleanup(true);

        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
	        // 解析参数
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }

        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }

```

这个方法也不是很长，前面的一些配置略过，主要看解析参数和`invokeStaticMain()`，顾名思义，这里的目的是通过反射调用静态的main方法，那么调用的是哪个类的`main`方法，我们看传入的参数是`args.startClass`，我们追溯参数的传递过程，发现之前提到的一系列`args`被封装进`ZygoteConnection.Arguments`类中，这一系列参数原本是

```
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1032,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
```

可以看到，唯一的类的全限定名是`com.android.server.SystemServer`，追述`args = new Arguments(argv)`的源码也可以找到答案：

```java
		Arguments(String args[]) throws IllegalArgumentException {
            parseArgs(args);
		}

        private void parseArgs(String args[])
                throws IllegalArgumentException {
            int curArg = 0;
            for (; curArg < args.length; curArg++) {
                String arg = args[curArg];

                if (arg.equals("--")) {
                    curArg++;
                    break;
                } else if (!arg.startsWith("--")) {
                    break;
                }
            }

            if (curArg == args.length) {
                throw new IllegalArgumentException("Missing classname argument to RuntimeInit!");
            }

			// startClass 为args的最后一个参数
            startClass = args[curArg++];
            startArgs = new String[args.length - curArg];
            System.arraycopy(args, curArg, startArgs, 0, startArgs.length);
        }
    }

```

### 6.3.6 RuntimeInit.invokeStaticMain

```java
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws Zygote.MethodAndArgsCaller {
        Class<?> cl;

        try {
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new Zygote.MethodAndArgsCaller(m, argv);
    }
复制代码
```

可以看到，这边走的就是正常的一个反射流程，会对`main`方法的参数、修饰符等校验，有问题抛出`RuntimeException`运行时异常，没问题就抛出一个`Zygote.MethodAndArgsCaller`，这一步的处理可以说是非常奇怪了，既然执行完毕为什么不直接invoke而是抛出异常呢，我们可以看到英文注释的大概解释，这个异常抛出在`ZygoteInit.main()`方法，执行了异常的run方法，其目的是清除设置中的所有堆栈帧，既然如此我们回到`ZygoteInit.main()`方法，果然：

```java
	try{
		...
	}catch (Zygote.MethodAndArgsCaller caller) {
        caller.run();
    }

```

抛出该异常调用了`caller.run()`方法，那我们看这里做了什么：

```java
    public static class MethodAndArgsCaller extends Exception
            implements Runnable {
        /** method to call */
        private final Method mMethod;

        /** argument array */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    }

```

可以看到，这里调用了`mMethod.invoke(null, new Object[] { mArgs })`，也就是执行了`SystemServer`的`main`方法。

### 6.3.7 SystemServer.main

```java
    public static void main(String[] args) {
        new SystemServer().run();
    }

		try {
			...
			//主线程在当前进程
			Looper.prepareMainLooper();
			...
            // 初始化系统上下文
            createSystemContext();
            
            // 1.创建SystemServiceManager
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setRuntimeRestarted(mRuntimeRestart);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
            // Prepare the thread pool for init tasks that can be parallelized
            SystemServerInitThreadPool.get();
		} finally {
			traceEnd();
		}
		...
        try {
            traceBeginAndSlog("StartServices");
			//2.开启一些重要的系统服务
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
            SystemServerInitThreadPool.shutdown();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            traceEnd();
        }
        ...
        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
```

可以看到，这边主要的工作，创建主线程`Looper`循环器，初始化系统上下文，最关键的还是后面的开启一系列的服务，这些服务就包括了系统服务，跟进`sartBootstrapService()`看看是如何初始化系统服务的。

### 6.3.8 SystemServer.startBootstrapServices

```java
		...
		SystemServerInitThreadPool.get().submit(SystemConfig::getInstance, TAG_SYSTEM_CONFIG);
		// 阻塞等待与installd建立socket通道
		Installer installer = mSystemServiceManager.startService(Installer.class);
		// 1. 在这里开启了Ams
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        ...

		// 2.在这里开启了PowerManagerService
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
		
		// 3.Ams 初始化PowerManager
        mActivityManagerService.initPowerManagement();
        
        // Bring up recovery system in case a rescue party needs a reboot
        if (!SystemProperties.getBoolean("config.disable_noncore", false)) {
            traceBeginAndSlog("StartRecoverySystemService");
            // 
            mSystemServiceManager.startService(RecoverySystemService.class);
            traceEnd();
        }
		
		mSystemServiceManager.startService(LightsService.class);
		
		mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
		
		mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();
        
		mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
		
		mSystemServiceManager.startService(new OverlayManagerService(mSystemContext, installer));
		
复制代码
```

可以看见启动了很多的系统服务，其中很多的服务目前也不是很了解，主要的是`ActivityMangerService`和`PackageManagerService`等主要管理系统的一些服务。自此，Android系统的`SystemServer`已经启动，并且其相关的各种系统服务已经启动，`Ams`等重要系统服务也均已开启。`Ams`是贯穿Android系统的核心服务，负责系统四大组件的启动、切换及其生命周期管理和调度等工作，各个应用程序App作为独立进程需要通过`Binder`与`Ams`进行通信，而`Ams`在`SystemServer`中通过`LocalSocket`与`Zygote`进行通信。不管是对于系统运行的原理，还是各组件的调度过程，理解`Ams`的工作原理十分重要，接下来将以此为主线，分别分析`Ams`、四大组件等的启动流程和生命周期的管理等。

## 6.4 面试题
![image-20210413103331499](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210413103331.png)
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303101142266.png)


# 7 SystemServer进程
http://gityuan.com/2016/02/14/android-system-server/

http://gityuan.com/2016/02/20/android-system-server-2/

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303071449416.png)

## 7.1 流程
![system_server_boot_process](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407181156.jpg)


![image-20210407181247452](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407181247.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303101422633.png)

SystemServer 的主要的作用是启动各种系统服务，比如 ActivityManagerService，PackageManagerService，WindowManagerService 以及硬件相关的 Service 等服务，我们平时熟知的各种系统服务其实都是在 SystemServer 进程中启动的，这些服务都运行在同一进程（即 SystemServer 进程）的不同线程中，而当我们的应用需要使用各种系统服务的时候其实也是通过与 SystemServer 进程通讯获取各种服务对象的句柄进而执行相应的操作的。在所有的服务启动完成后，会调用各服务的 service.systemReady(…) 来通知各对应的服务，系统已经就绪。

## 7.2 主要工作
system_server进程，从源码角度划分为引导服务、核心服务、其他服务3类。 以下这些系统服务的注册过程, 见[Android系统服务的注册方式](http://gityuan.com/2016/10/01/system_service_common/)

1.  **引导服务**(7个)：ActivityManagerService、PowerManagerService、LightsService、DisplayManagerService、PackageManagerService、UserManagerService、SensorService；
2.  **核心服务**(3个)：BatteryService、UsageStatsService、WebViewUpdateService；
3.  **其他服务**(70个+)：AlarmManagerService、VibratorService等。

```Java
private void run() {
    //当系统时间比1970年更早，就设置当前系统时间为1970年
    if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
        SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
    }

    //变更虚拟机的库文件，对于Android 6.0默认采用的是libart.so
    SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

    if (SamplingProfilerIntegration.isEnabled()) {
        ...
    }

    //清除vm内存增长上限，由于启动过程需要较多的虚拟机内存空间
    VMRuntime.getRuntime().clearGrowthLimit();

    //设置内存的可能有效使用率为0.8
    VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
    // 针对部分设备依赖于运行时就产生指纹信息，因此需要在开机完成前已经定义
    Build.ensureFingerprintProperty();

    //访问环境变量前，需要明确地指定用户
    Environment.setUserRequired(true);

    //确保当前系统进程的binder调用，总是运行在前台优先级(foreground priority)
    BinderInternal.disableBackgroundScheduling(true);
    android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_FOREGROUND);
    android.os.Process.setCanSelfBackground(false);

    // 主线程looper就在当前线程运行
    Looper.prepareMainLooper();

    //加载android_servers.so库，该库包含的源码在frameworks/base/services/目录下
    System.loadLibrary("android_servers");

    //检测上次关机过程是否失败，该方法可能不会返回[见小节1.2.1]
    performPendingShutdown();

    //初始化系统上下文 【见小节1.3】
    createSystemContext();

    //创建系统服务管理
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    //将mSystemServiceManager添加到本地服务的成员sLocalServiceObjects
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);


    //启动各种系统服务
    try {
        startBootstrapServices(); // 启动引导服务【见小节1.4】
        startCoreServices();      // 启动核心服务【见小节1.5】
        startOtherServices();     // 启动其他服务【见小节1.6】
    } catch (Throwable ex) {
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    }

    //用于debug版本，将log事件不断循环地输出到dropbox（用于分析）
    if (StrictMode.conditionallyEnableDebugLogging()) {
        Slog.i(TAG, "Enabled StrictMode for system server main thread.");
    }
    //一直循环执行
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

### 7.2.1 createSystemContext

```java
private void createSystemContext() {
    //创建system_server进程的上下文信息
    ActivityThread activityThread = ActivityThread.systemMain();
    mSystemContext = activityThread.getSystemContext();
    //设置主题
    mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
}
```

[理解Application创建过程](http://gityuan.com/2017/04/02/android-application/)已介绍过createSystemContext()过程， 该过程会创建对象有ActivityThread，Instrumentation, ContextImpl，LoadedApk，Application。

### 7.2.2 startBootstrapServices

```java
private void startBootstrapServices() {
    //阻塞等待与installd建立socket通道
    Installer installer = mSystemServiceManager.startService(Installer.class);

    //启动服务ActivityManagerService
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);

    //启动服务PowerManagerService
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

    //初始化power management
    mActivityManagerService.initPowerManagement();

    //启动服务LightsService
    mSystemServiceManager.startService(LightsService.class);

    //启动服务DisplayManagerService
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    //Phase100: 在初始化package manager之前，需要默认的显示.
    mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

    //当设备正在加密时，仅运行核心
    String cryptState = SystemProperties.get("vold.decrypt");
    if (ENCRYPTING_STATE.equals(cryptState)) {
        mOnlyCore = true;
    } else if (ENCRYPTED_STATE.equals(cryptState)) {
        mOnlyCore = true;
    }

    //启动服务PackageManagerService
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();

    //启动服务UserManagerService，新建目录/data/user/
    ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

    AttributeCache.init(mSystemContext);

    //设置AMS
    mActivityManagerService.setSystemProcess();

    //启动传感器服务
    startSensorService();
}
```

该方法所创建的服务：ActivityManagerService, PowerManagerService, LightsService, DisplayManagerService， PackageManagerService， UserManagerService， sensor服务.

### 7.2.3 startCoreServices

```java
private void startCoreServices() {
    //启动服务BatteryService，用于统计电池电量，需要LightService.
    mSystemServiceManager.startService(BatteryService.class);

    //启动服务UsageStatsService，用于统计应用使用情况
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));

    mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

    //启动服务WebViewUpdateService
    mSystemServiceManager.startService(WebViewUpdateService.class);
}
```

启动服务BatteryService，UsageStatsService，WebViewUpdateService。

### 7.2.4  startOtherServices

该方法比较长，有近千行代码，逻辑很简单，主要是启动一系列的服务，这里就不具体列举源码了，在第四节直接对其中的服务进行一个简单分类。

```java
    private void startOtherServices() {
        ...
        SystemConfig.getInstance();
        mContentResolver = context.getContentResolver(); // resolver
        ...
        mActivityManagerService.installSystemProviders(); //provider
        mSystemServiceManager.startService(AlarmManagerService.class); // alarm
        // watchdog
        watchdog.init(context, mActivityManagerService); 
        inputManager = new InputManagerService(context); // input
        wm = WindowManagerService.main(...); // window
        inputManager.start();  //启动input
        mDisplayManagerService.windowManagerAndInputReady();
        ...
        mSystemServiceManager.startService(MOUNT_SERVICE_CLASS); // mount
        mPackageManagerService.performBootDexOpt();  // dexopt操作
        ActivityManagerNative.getDefault().showBootMessage(...); //显示启动界面
        ...
        statusBar = new StatusBarManagerService(context, wm); //statusBar
        //dropbox
        ServiceManager.addService(Context.DROPBOX_SERVICE,
                    new DropBoxManagerService(context, new File("/data/system/dropbox")));
         mSystemServiceManager.startService(JobSchedulerService.class); //JobScheduler
         lockSettings.systemReady(); //lockSettings

        //phase480 和phase500
        mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
        mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
        ...
        // 准备好window, power, package, display服务
        wm.systemReady();
        mPowerManagerService.systemReady(...);
        mPackageManagerService.systemReady();
        mDisplayManagerService.systemReady(...);
        
        //重头戏[见小节2.1]
        mActivityManagerService.systemReady(new Runnable() {
            public void run() {
              ...
            }
        });
    }
```

SystemServer启动各种服务中最后的一个环节便是AMS.systemReady()，详见[ActivityManagerService启动过程](http://gityuan.com/2016/02/21/activity-manager-service/).

## 7.3 SystemServiceManager
```plantuml
class SystemServer {
	SystemServiceManager mSystemServiceManager;
	PowerManagerService mPowerManagerService;  
	ActivityManagerService mActivityManagerService;  
	PackageManagerService mPackageManagerService;
}
```

## 7.4 面试题
```java
1.SystemServer是如何被fork出来的 
2.SystemServer做了些什么事情
3.SystemServiceManager是负责什么的？ 
4.SystemServiceManager是如何创建的？
5.ServiceManager是负责什么的？
6.启动服务有几种方式？他们之间有什么区别？
7.SystemServiceManager和ServiceManager有什么区别？
8.LocalService是负责什么工作的？
9.SystemServer是服务端进程，那么谁是客户端进程？他们是如何通信的？内部机制是什么？（Binder相关的问题）
```

# 8 ServiceManager进程
[[ServiceManager]]

```plantuml
interface IServiceManager extends IInterface{
IBinder getService(String name);
void addService(String name, IBinder service, boolean allowIsolated);
}

class ServiceManagerNative extends Binder implements IServiceManager{
	IServiceManager asInterface(IBinder obj);
	boolean onTransact(int code, Parcel data, Parcel reply, int flags);
}

class ServiceManagerProxy implements IServiceManager {

}
```

## 8.1 面试题
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303101137773.png)

# 9 Android系统服务的注册方式
http://gityuan.com/2016/10/01/system_service_common/
https://blog.csdn.net/b1480521874/article/details/79422909


## 9.1 概述
启动启动过程有采用过两种不同的方式来注册系统服务：
-   ServiceManager的addService()
-   SystemServiceManager的startService()

其核心都是向ServiceManager进程注册binder服务，但功能略有不同，下面从源码角度详加说明。

## 9.2 总结
方式1. ServiceManager.addService():

-   功能：向ServiceManager注册该服务.
-   特点：服务往往直接或间接继承于Binder服务；
-   举例：input, window, package；

方式2. SystemServiceManager.startService:

-   功能：
    -   创建服务对象；
    -   执行该服务的onStart()方法；该方法会执行上面的SM.addService()；
    -   根据启动到不同的阶段会回调onBootPhase()方法；
    -   另外，还有多用户模式下用户状态的改变也会有回调方法；例如onStartUser();
-   特点：服务往往自身或内部类继承于SystemService；
-   举例：power, activity；

两种方式真正注册服务的过程都会调用到ServiceManager.addService()方法. 对于方式2多了**一个服务对象创建以及 根据不同启动阶段采用不同的动作的过程**。可以理解为方式2比方式1的功能更丰富。



# 10 面试题

## 10.1 点击桌面图标的启动过程？涉及的进程和组件
https://www.kancloud.cn/alex_wsc/androids/477726

## 10.2 为什么SystemServer进程与Zygote进程通讯采用Socket而不是Binder
在Android系统中，SystemServer进程与Zygote进程之间需要进行跨进程通讯，其中，SystemServer进程需要向Zygote进程发出请求，请求Zygote进程创建一个新进程。这个请求的目的是为了启动新的应用程序或服务。

Android系统中的Binder机制是一种高效的跨进程通讯机制，但是在SystemServer进程与Zygote进程之间通讯时，Socket机制更加适合。这是因为：

1.  简单易用：Socket机制比Binder机制更易于使用和实现。
    
2.  可靠性高：Socket机制是面向连接的通讯方式，实现起来比较稳定，即使发生网络异常或者通讯故障，也可以进行重连。
    
3.  兼容性好：Socket机制可以跨平台，包括不同的系统或不同的进程之间都可以通讯。
    
4.  处理复杂数据方便：Socket机制可以直接传输二进制数据，处理复杂数据类型比Binder更加方便。
    

综上所述，虽然Binder机制是Android系统中推荐的跨进程通讯方式，但是在SystemServer进程与Zygote进程之间通讯时，Socket机制更加适合。

## 10.3 Application#onCreate里面可以使用binder服务么(可以)？Application的binder机制是何时启动的（zygote在fork好应用进程后，会给应用启动binder机制）？binder机制启动的几个关键步骤。



## 10.4 本章重点讲解系统核心进程，以及一些关键的系统服务的启动原理和工作原理相关的面试内容。

-  2-1 谈谈对zygote的理解 (17:27)**试看**
-  2-2 说说Android系统的启动 (15:38)**试看**
-  2-3 你知道怎么添加一个系统服务吗？ (16:57)
-  2-4 系统服务和bind的应用服务有什么区别？ (07:11)
-  2-5 ServiceManager启动和工作原理是怎样的？


## 10.5 Android中LocalServerSocket是干什么用的



## 10.6 zygote面试题


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051518061.png)




setContentView在大厂面试中必问问题
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051605969.png)

PhoneWindow ----->LayoutInflater
问题一：
区分系统 view 和自定义
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051611953.png)

问题二：
都不能删除

问题三：
抛异常、编译不通过
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051617819.png)


问题四：
栈会溢出，出现 OOM，优化布局

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051623745.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051630166.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051630805.png)











C/C++代码：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051958562.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303051959776.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303052001623.png)


创建新进程：socket 消息，zygote 服务器等待（一个 while 的死循环等待 socket 消息）

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303052008922.png)






https://www.bilibili.com/video/BV1ug411X7kD?p=5&spm_id_from=pageDriver&vd_source=f089cad7f2e400622e91946932061cb9
问题1：
app运行，需要：虚拟机，资源，SDK代码
fork 是复制



问题2：
socket->效率低，socket通信，一个一个执行，不安全

如果是 socket，zygote fork 之后每个进程都有socket

binder：多线程、效率、安全
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303052028477.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303052033448.png)

问题3：


问题4：


问题5：


问题6：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303052050117.png)
