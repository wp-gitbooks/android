# 参考

http://gityuan.com/2016/03/26/app-process-create/



# Android进程

http://gityuan.com/2016/03/26/app-process-create/

https://mp.weixin.qq.com/s/k1cd4s18tmF4kqHj4unOTA



http://gityuan.com/tags/#PMS

![image-20210412170724052](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210412170724.png)

## 概述

### 父进程

http://gityuan.com/2015/12/19/android-process-category/



### fork()

fork()采用copy on write技术，这是linux创建进程的标准方法，调用一次，返回两次，返回值有3种类型。

- 父进程中，fork返回新创建的子进程的pid;
- 子进程中，fork返回0；
- 当出现错误时，fork返回负数。（当进程数超过上限或者系统内存不足时会出错）

fork()的主要工作是寻找空闲的进程号pid，然后从父进程拷贝进程信息，例如数据段和代码段，fork()后子进程要执行的代码等。 Zygote进程是所有Android进程的母体，包括system_server和各个App进程。zygote利用fork()方法生成新进程，对于新进程A复用Zygote进程本身的资源，再加上新进程A相关的资源，构成新的应用进程A。其中下图中Zygote进程的libc、vm、preloaded classes、preloaded resources是如何生成的，可查看另一个文章[Android系统启动-zygote篇](http://gityuan.com/2016/02/13/android-zygote/#preload)，见下图：

![zygote_fork](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210316154830.jpg)

copy-on-write过程：当父子进程任一方修改内存数据时（这是on-write时机），才发生缺页中断，从而分配新的物理内存（这是copy操作）。

copy-on-write原理：写时拷贝是指子进程与父进程的页表都所指向同一个块物理内存，fork过程只拷贝父进程的页表，并标记这些页表是只读的。父子进程共用同一份物理内存，如果父子进程任一方想要修改这块物理内存，那么会触发缺页异常(page fault)，Linux收到该中断便会创建新的物理内存，并将两个物理内存标记设置为可写状态，从而父子进程都有各自独立的物理内存







## 创建流程图

http://gityuan.com/2016/03/26/app-process-create/

![start_app_process](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210310210944.jpg)



## 杀进程的实现原理

http://gityuan.com/2016/04/16/kill-signal/



## 进程绝杀技--forceStop

![am_force_stop](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210310211436.jpg)





## 四大组件与进程启动的关系

![component_process](http://gityuan.com/images/process/component_process.jpg)



![app_process_ipc](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210310211229.jpg)



![start_process](http://gityuan.com/images/process/start_process.jpg)



# Android线程

## Android线程创建流程

http://gityuan.com/2016/09/24/android-thread/



# Android进程生命周期与ADJ

http://gityuan.com/2015/10/01/process-lifecycle/

http://gityuan.com/2016/08/07/android-adj/

http://gityuan.com/2018/05/19/android-process-adj/



# Android进程优先级

http://gityuan.com/2016/09/17/android-lowmemorykiller/





# App保活

http://gityuan.com/2018/02/24/process-keep-forever/

http://www.52im.net/forum.php?mod=viewthread&tid=2893&highlight=%B1%A3%BB%EE

http://weishu.me/2021/01/25/another-keep-alive-method/



# 面试题

## Service与Activity之间通信的几种方式

- 通过Binder对象
- 通过broadcast(广播)的形式



## 如何保证service在后台不被Kill

一、onStartCommand方法，返回START_STICKY

1. START_STICKY 在运行onStartCommand后service进程被kill后，那将保留在开始状态，但是不保留那些传入的intent。不久后service就会再次尝试重新创建，因为保留在开始状态，在创建 service后将保证调用onstartCommand。如果没有传递任何开始命令给service，那将获取到null的intent。
2. START_NOT_STICKY 在运行onStartCommand后service进程被kill后，并且没有新的intent传递给它。Service将移出开始状态，并且直到新的明显的方法（startService）调用才重新创建。因为如果没有传递任何未决定的intent那么service是不会启动，也就是期间onstartCommand不会接收到任何null的intent。
3. START_REDELIVER_INTENT 在运行onStartCommand后service进程被kill后，系统将会再次启动service，并传入最后一个intent给onstartCommand。直到调用stopSelf(int)才停止传递intent。如果在被kill后还有未处理好的intent，那被kill后服务还是会自动启动。因此onstartCommand不会接收到任何null的intent。

二、提升service进程优先级

Android中的进程是托管的，当系统进程空间紧张的时候，会依照优先级自动进行进程的回收。Android将进程分为6个等级,它们按优先级顺序由高到低依次是:

1. 前台进程( FOREGROUND_APP)
2. 可视进程(VISIBLE_APP )
3. 次要服务进程(SECONDARY_SERVER )
4. 后台进程 (HIDDEN_APP)
5. 内容供应节点(CONTENT_PROVIDER)
6. 空进程(EMPTY_APP)

当service运行在低内存的环境时，将会kill掉一些存在的进程。因此进程的优先级将会很重要，可以使用startForeground 将service放到前台状态。这样在低内存时被kill的几率会低一些。

三、onDestroy方法里重启service

service +broadcast 方式，就是当service走ondestory的时候，发送一个自定义的广播，当收到广播的时候，重新启动service；

四、Application加上Persistent属性

五、监听系统广播判断Service状态

通过系统的一些广播，比如：手机重启、界面唤醒、应用状态改变等等监听并捕获到，然后判断我们的Service是否还存活，别忘记加权限啊。



## 如何判断一个 APP 在前台还是后台？



## 如何做应用保活？

我们可以**利用SyncAdapter提高进程优先级**，它是Android系统提供一个**账号同步机制**，它属于**核心进程级别**，而使用了SyncAdapter的进程优先级本身也会提高，使用方式请Google，关联SyncAdapter后，进程的**优先级变为1**，仅低于前台正在运行的进程，因此可以**降低应用被系统杀掉的概率**。





## 进程保活（不死进程）

此处延伸：进程的优先级是什么

当前业界的Android进程保活手段主要分为** 黑、白、灰 **三种，其大致的实现思路如下：

黑色保活：不同的app进程，用广播相互唤醒（包括利用系统提供的广播进行唤醒）

白色保活：启动前台Service

灰色保活：利用系统的漏洞启动前台Service

黑色保活

所谓黑色保活，就是利用不同的app进程使用广播来进行相互唤醒。举个3个比较常见的场景：

场景1：开机，网络切换、拍照、拍视频时候，利用系统产生的广播唤醒app

场景2：接入第三方SDK也会唤醒相应的app进程，如微信sdk会唤醒微信，支付宝sdk会唤醒支付宝。由此发散开去，就会直接触发了下面的 场景3

场景3：假如你手机里装了支付宝、淘宝、天猫、UC等阿里系的app，那么你打开任意一个阿里系的app后，有可能就顺便把其他阿里系的app给唤醒了。（只是拿阿里打个比方，其实BAT系都差不多）

白色保活

白色保活手段非常简单，就是调用系统api启动一个前台的Service进程，这样会在系统的通知栏生成一个Notification，用来让用户知道有这样一个app在运行着，哪怕当前的app退到了后台。如下方的LBE和QQ音乐这样：

灰色保活

灰色保活，这种保活手段是应用范围最广泛。它是利用系统的漏洞来启动一个前台的Service进程，与普通的启动方式区别在于，它不会在系统通知栏处出现一个Notification，看起来就如同运行着一个后台Service进程一样。这样做带来的好处就是，用户无法察觉到你运行着一个前台进程（因为看不到Notification）,但你的进程优先级又是高于普通后台进程的。那么如何利用系统的漏洞呢，大致的实现思路和代码如下：

思路一：API < 18，启动前台Service时直接传入new Notification()；

思路二：API >= 18，同时启动两个id相同的前台Service，然后再将后启动的Service做stop处理

熟悉Android系统的童鞋都知道，系统出于体验和性能上的考虑，app在退到后台时系统并不会真正的kill掉这个进程，而是将其缓存起来。打开的应用越多，后台缓存的进程也越多。在系统内存不足的情况下，系统开始依据自身的一套进程回收机制来判断要kill掉哪些进程，以腾出内存来供给需要的app。这套杀进程回收内存的机制就叫 Low Memory Killer ，它是基于Linux内核的 OOM Killer（Out-Of-Memory killer）机制诞生。

进程的重要性，划分5级：

前台进程 (Foreground process)

可见进程 (Visible process)

服务进程 (Service process)

后台进程 (Background process)

空进程 (Empty process)



了解完 Low Memory Killer，再科普一下oom_adj。什么是oom_adj？它是linux内核分配给每个系统进程的一个值，代表进程的优先级，进程回收机制就是根据这个优先级来决定是否进行回收。对于oom_adj的作用，你只需要记住以下几点即可：

进程的oom_adj越大，表示此进程优先级越低，越容易被杀回收；越小，表示进程优先级越高，越不容易被杀回收

普通app进程的oom_adj>=0,系统进程的oom_adj才可能<0

有些手机厂商把这些知名的app放入了自己的白名单中，保证了进程不死来提高用户体验（如微信、QQ、陌陌都在小米的白名单中）。如果从白名单中移除，他们终究还是和普通app一样躲避不了被杀的命运，为了尽量避免被杀，还是老老实实去做好优化工作吧。

所以，进程保活的根本方案终究还是回到了性能优化上，进程永生不死终究是个彻头彻尾的伪命题！





作者：王培921223
链接：https://www.jianshu.com/p/7661c292195a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



## 进程是怎么启动的

