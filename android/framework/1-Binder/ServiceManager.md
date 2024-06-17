# ServiceManager

ServiceManager本身工作相对简单，其功能：查询和注册服务

ServiceManager是由init进程启动

![class_java_binder](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519142440.jpg)


-   **ServiceManager：** 通过getIServiceManager方法获取的是ServiceManagerProxy对象； ServiceManager的addService, getService实际工作都交由ServiceManagerProxy的相应方法来处理；
  
-   **ServiceManagerProxy：** 其成员变量mRemote指向BinderProxy对象，ServiceManagerProxy的addService, getService方法最终是交由mRemote来完成。
  
-   **ServiceManagerNative**：其方法asInterface()返回的是ServiceManagerProxy对象，ServiceManager便是借助ServiceManagerNative类来找到ServiceManagerProxy；
  
-   **Binder：** 其成员变量mObject和方法execTransact()用于native方法
  
-   **BinderInternal：** 内部有一个GcWatcher类，用于处理和调试与Binder相关的垃圾回收。
  
-   **IBinder：** 接口中常量FLAG_ONEWAY：客户端利用binder跟服务端通信是阻塞式的，但如果设置了FLAG_ONEWAY，这成为非阻塞的调用方式，客户端能立即返回，服务端采用回调方式来通知客户端完成情况。另外IBinder接口有一个内部接口DeathDecipient(死亡通告)


## 启动结构图

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528144043.jpeg)

1.  ServiceManager 分为 framework 层和 native 层，framework 层只是对 native 层进行了封装方便调用，图上展示的是 native 层的 ServiceManager 启动过程。
   
2.  ServiceManager 的启动是系统在开机时，init 进程解析 init.rc 文件调用 service_manager.c 中的 main() 方法入口启动的。 native 层有一个 binder.c 封装了一些与 Binder 驱动交互的方法。
   
3.  ServiceManager 的启动分为三步，首先打开驱动创建全局链表 binder_procs，然后将自己当前进程信息保存到 binder_procs 链表，最后开启 loop 不断的处理共享内存中的数据，并处理 BR_xxx 命令（ioctl 的命令，BR 可以理解为 binder reply 驱动处理完的响应）



## 启动流程图
启动过程主要以下几个阶段：
1.  打开binder驱动：binder_open；
2.  注册成为binder服务的大管家(注册为Binder上下文管理者)：binder_become_context_manager；
3.  进入无限循环，处理client端发来的请求：binder_loop；


![create_servicemanager](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210309223430.jpg)

### 总结
ServiceManger集中管理系统内的所有服务，通过权限控制进程是否有权注册服务,通过字符串名称来查找对应的Service; 由于ServiceManger进程建立跟所有向其注册服务的死亡通知, 那么当服务所在进程死亡后, 会只需告知ServiceManager. 每个Client通过查询ServiceManager可获取Server进程的情况，降低所有Client进程直接检测会导致负载过重。

**ServiceManager启动流程：**

1. 打开binder驱动，并调用mmap()方法分配**128k的内存映射空间**：binder_open();
2. 通知binder驱动使其成为守护进程：binder_become_context_manager()；
3. 验证selinux权限，判断进程是否有权注册或查看指定服务；
4. 进入循环状态，等待Client端的请求：binder_loop()。
5. 注册服务的过程，根据服务名称，但同一个服务已注册，重新注册前会先移除之前的注册信息；
6. 死亡通知: 当binder所在进程死亡后,会调用binder_release方法,然后调用binder_node_release.这个过程便会发出死亡通知的回调.

ServiceManager最核心的两个功能为查询和注册服务：

- 注册服务：记录服务名和handle信息，保存到svclist列表；
- 查询服务：根据服务名查询相应的的handle信息



## 功能

**ServiceManager启动流程：**

1.  打开binder驱动，并调用mmap()方法分配128k的内存映射空间：binder_open();
    
2.  通知binder驱动使其成为守护进程：binder_become_context_manager()；
    
3.  验证selinux权限，判断进程是否有权注册或查看指定服务；
    
4.  进入循环状态，等待Client端的请求：binder_loop()。
    
5.  注册服务的过程，根据服务名称，但同一个服务已注册，重新注册前会先移除之前的注册信息；
    
6.  死亡通知: 当binder所在进程死亡后,会调用binder_release方法,然后调用binder_node_release.这个过程便会发出死亡通知的回调.
    

ServiceManager最核心的两个功能为查询和注册服务：

-   注册服务：记录服务名和handle信息，保存到svclist列表；
    
-   查询服务：根据服务名查询相应的的handle信息
    

### ServiceManager注册服务

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528144203.jpeg)

-   注册 MediaPlayerService 服务端，我们通过 ServiceManager 的 addService() 方法来注册服务。
    
-   首先 ServiceManager 向 Binder 驱动发送 BC_TRANSACTION 命令（ioctl 的命令，BC 可以理解为 binder client 客户端发过来的请求命令）携带 ADD_SERVICE_TRANSACTION 命令，同时注册服务的线程进入等待状态 waitForResponse()。 Binder 驱动收到请求命令向 ServiceManager 的 todo 队列里面添加一条注册服务的事务。事务的任务就是创建服务端进程 binder_node 信息并插入到 binder_procs 链表中。
    
-   事务处理完之后发送 BR_TRANSACTION 命令，ServiceManager 收到命令后向 svcinfo 列表中添加已经注册的服务。最后发送 BR_REPLY 命令唤醒等待的线程，通知注册成功。
    

### ServiceManager获取服务

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528144227.jpeg)

-   获取服务的过程与注册类似，相反的过程。通过 ServiceManager 的 getService() 方法来注册服务。
    
-   首先 ServiceManager 向 Binder 驱动发送 BC_TRANSACTION 命令携带 CHECK_SERVICE_TRANSACTION 命令，同时获取服务的线程进入等待状态 waitForResponse()。
    
-   Binder 驱动收到请求命令向 ServiceManager 的发送 BC_TRANSACTION 查询已注册的服务，查询到直接响应 BR_REPLY 唤醒等待的线程。若查询不到将与 binder_procs 链表中的服务进行一次通讯再响应
    




## 获取ServiceManager流程图

![get_servicemanager](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210309223810.jpg)

## 注册服务(addService)

### 类图

![add_media_player_service](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210309223942.png)

### 时序图

![addService](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210309224009.jpg)

### ProcessState初始化

ProcessState.cpp

```
ProcessState::ProcessState()
    : mDriverFD(open_driver()) // 打开Binder驱动【见小节2.3】
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        //采用内存映射函数mmap，给binder分配一块虚拟地址空间【见小节2.4】
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            close(mDriverFD); //没有足够空间分配给/dev/binder,则关闭驱动
            mDriverFD = -1;
        }
    }
}
```

- `ProcessState`的单例模式的惟一性，因此一个进程只打开binder设备一次,其中ProcessState的成员变量`mDriverFD`记录binder驱动的fd，用于访问binder设备。
- `BINDER_VM_SIZE = (1*1024*1024) - (4096 *2)`, binder分配的默认内存大小为1M-8k。
- `DEFAULT_MAX_BINDER_THREADS = 15`，binder默认的最大可并发访问的线程数为16

ProcessState采用单例模式，**保证每一个进程**都只打开一次Binder Driver。





## 获取服务(getService)

### 类图

![get_media_player_service](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210309225017.png)



# 参考
https://wangkuiwu.github.io/2014/09/03/Binder-ServiceManager-Daemon/
https://blog.csdn.net/qq_38366777/article/details/108221171