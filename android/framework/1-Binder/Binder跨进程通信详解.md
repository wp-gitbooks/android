# Binder原理



![ServiceManager](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210314220702.jpg)



# Binder IPC原理

Binder IPC 正是基于内存映射（mmap）来实现的，但是 mmap() 通常是用在有物理介质的文件系统上的。

比如进程中的用户区域是不能直接和物理设备打交道的，如果想要把磁盘上的数据读取到进程的用户区域，需要两次拷贝（磁盘–>内核空间–>用户空间）；通常在这种场景下 mmap() 就能发挥作用，通过在物理介质和用户空间之间建立映射，减少数据的拷贝次数，用内存读写取代I/O读写，提高文件读取效率。

而 Binder 并不存在物理介质，因此 Binder 驱动使用 mmap() 并不是为了在物理介质和用户空间之间建立映射，而是用来在内核空间创建数据接收的缓存空间。

一次完整的 Binder IPC 通信过程通常是这样：

1. 首先 Binder 驱动在内核空间创建一个数据接收缓存区；
2. 接着在内核空间开辟一块内核缓存区，建立**内核缓存区**和**内核中数据接收缓存区**之间的映射关系，以及**内核中数据接收缓存区**和**接收进程用户空间地址**的映射关系；
3. 发送方进程通过系统调用 copy_from_user() 将数据 copy 到内核中的**内核缓存区**，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信

![Binder IPC 原理](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407134121.png)





# Binder跨进程原理

## 简单

http://gityuan.com/2016/09/04/binder-start-service/

Client进程通过RPC(Remote Procedure Call Protocol)与Server通信，可以简单地划分为三层，驱动层、IPC层、业务层。`demo()`便是Client端和Server共同协商好的统一方法；handle、RPC数据、代码、协议这4项组成了IPC层的数据，通过IPC层进行数据传输；而真正在Client和Server两端建立通信的基础设施便是Binder Driver。

![binder_ipc](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210309222830.jpg)

### 通信模型

介绍完 Binder IPC 的底层通信原理，接下来我们看看实现层面是如何设计的。

一次完整的进程间通信必然至少包含两个进程，通常我们称通信的双方分别为客户端进程（Client）和服务端进程（Server），由于进程隔离机制的存在，通信双方必然需要借助 Binder 来实现



#### Client/Server/ServiceManager/驱动

前面我们介绍过，Binder 是基于 C/S 架构的。由一系列的组件组成，包括 Client、Server、ServiceManager、Binder 驱动。其中 Client、Server、Service Manager 运行在用户空间，Binder 驱动运行在内核空间。其中 Service Manager 和 Binder 驱动由系统提供，而 Client、Server 由应用程序来实现。Client、Server 和 ServiceManager 均是通过系统调用 open、mmap 和 ioctl 来访问设备文件 /dev/binder，从而实现与 Binder 驱动的交互来间接的实现跨进程通信。



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407134322.png)



1. SM建立(建立通信录)；首先有一个进程向驱动提出申请为SM；驱动同意之后，SM进程负责管理Service（注意这里是Service而不是Server，因为如果通信过程反过来的话，那么原来的客户端Client也会成为服务端Server）不过这时候通信录还是空的，一个号码都没有。
2. 各个Server向SM注册(完善通信录)；每个Server端进程启动之后，向SM报告，我是zhangsan, 要找我请返回0x1234(这个地址没有实际意义，类比)；其他Server进程依次如此；这样SM就建立了一张表，对应着各个Server的名字和地址；就好比B与A见面了，说存个我的号码吧，以后找我拨打10086；
3. Client想要与Server通信，首先询问SM；请告诉我如何联系zhangsan，SM收到后给他一个号码0x1234；Client收到之后，开心滴用这个号码拨通了Server的电话，于是就开始通信了

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407142917.png)

Client、Server、ServiceManager、Binder 驱动这几个组件在通信过程中扮演的角色就如同互联网中服务器（Server）、客户端（Client）、DNS域名服务器（ServiceManager）以及路由器（Binder 驱动）之前的关系。

通常我们访问一个网页的步骤是这样的：首先在浏览器输入一个地址，如 [www.google.com](http://www.google.com/) 然后按下回车键。但是并没有办法通过域名地址直接找到我们要访问的服务器，因此需要首先访问 DNS 域名服务器，域名服务器中保存了 [www.google.com](http://www.google.com/) 对应的 ip 地址 10.249.23.13，然后通过这个 ip 地址才能访问到 [www.google.com](http://www.google.com/) 对应的服务器。

![互联网通信模型](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407134403.png)







![binder_protocol](http://gityuan.com/images/binder/binder_dev/binder_transaction_ipc.jpg)

### 通信过程 (调用过程)

![binder_protocol](http://gityuan.com/images/binder/binder_dev/binder_protocol.jpg)



至此，我们大致能总结出 Binder 通信过程：

1. 首先，一个进程使用 BINDER_SET_CONTEXT_MGR 命令通过 Binder 驱动将自己注册成为 ServiceManager；
2. Server 通过驱动向 ServiceManager 中注册 Binder（Server 中的 Binder 实体），表明可以对外提供服务。驱动为这个 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager，ServiceManger 将其填入查找表。
3. Client 通过名字，在 Binder 驱动的帮助下从 ServiceManager 中获取到对 Binder 实体的引用，通过这个引用就能实现和 Server 进程的通信。

我们看到整个通信过程都需要 Binder 驱动的接入。下图能更加直观的展现整个通信过程(为了进一步抽象通信过程以及呈现上的方便，下图我们忽略了 Binder 实体及其引用的概念)：

![Binder 通信模型](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407134554.png)



### Binder 通信中的代理模式

我们已经解释清楚 Client、Server 借助 Binder 驱动完成跨进程通信的实现机制了，但是还有个问题会让我们困惑。A 进程想要 B 进程中某个对象（object）是如何实现的呢？毕竟它们分属不同的进程，A 进程 没法直接使用 B 进程中的 object。

前面我们介绍过跨进程通信的过程都有 Binder 驱动的参与，因此在数据流经 Binder 驱动的时候驱动会对数据做一层转换。当 A 进程想要获取 B 进程中的 object 时，驱动并不会真的把 object 返回给 A，而是返回了一个跟 object 看起来一模一样的代理对象 objectProxy，这个 objectProxy 具有和 object 一摸一样的方法，但是这些方法并没有 B 进程中 object 对象那些方法的能力，这些方法只需要把把请求参数交给驱动即可。对于 A 进程来说和直接调用 object 中的方法是一样的。

当 Binder 驱动接收到 A 进程的消息后，发现这是个 objectProxy 就去查询自己维护的表单，一查发现这是 B 进程 object 的代理对象。于是就会去通知 B 进程调用 object 的方法，并要求 B 进程把返回结果发给自己。当驱动拿到 B 进程的返回结果后就会转发给 A 进程，一次通信就完成了

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210407134705.png)

首先，Server进程要向SM注册；告诉自己是谁，自己有什么能力；在这个场景就是Server告诉SM，它叫`zhangsan`，它有一个`object`对象，可以执行`add` 操作；于是SM建立了一张表：`zhangsan`这个名字对应进程Server;

然后Client向SM查询：我需要联系一个名字叫做`zhangsan`的进程里面的`object`对象；这时候关键来了：进程之间通信的数据都会经过运行在内核空间里面的驱动，驱动在数据流过的时候做了一点手脚，它并不会给Client进程返回一个真正的`object`对象，而是返回一个看起来跟`object`一模一样的代理对象`objectProxy`，这个`objectProxy`也有一个`add`方法，但是这个`add`方法没有Server进程里面`object`对象的`add`方法那个能力；`objectProxy`的`add`只是一个傀儡，它唯一做的事情就是把参数包装然后交给驱动。(这里我们简化了SM的流程，见下文)

但是Client进程并不知道驱动返回给它的对象动过手脚，毕竟伪装的太像了，如假包换。Client开开心心地拿着`objectProxy`对象然后调用`add`方法；我们说过，这个`add`什么也不做，直接把参数做一些包装然后直接转发给Binder驱动。

驱动收到这个消息，发现是这个`objectProxy`；一查表就明白了：我之前用`objectProxy`替换了`object`发送给Client了，它真正应该要访问的是`object`对象的`add`方法；于是Binder驱动通知Server进程，*调用你的object对象的`add`方法，然后把结果发给我*，Sever进程收到这个消息，照做之后将结果返回驱动，驱动然后把结果返回给`Client`进程；于是整个过程就完成了。

由于驱动返回的`objectProxy`与Server进程里面原始的`object`是如此相似，给人感觉好像是**直接把Server进程里面的对象object传递到了Client进程**；因此，我们可以说**Binder对象是可以进行跨进程传递的对象**



## 详细

### 模型原理图

![示意图](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329095146.png)



![Alt text](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329170821.png)





### 模型组成角色说明

![示意图](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329095215.png)


### `Binder`驱动的作用 & 原理

![示意图](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329095258.png)



![示意图](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329095320.png)



### 模型原理步骤说明

![示意图](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329095413.png)





### 额外说明

#### 说明1：`Client`进程、`Server`进程 & `Service Manager` 进程之间的交互 都必须通过`Binder`驱动（使用 `open` 和 `ioctl`文件操作函数），而非直接交互

原因：

1. `Client`进程、`Server`进程 & `Service Manager`进程属于进程空间的用户空间，不可进行进程间交互
2. `Binder`驱动 属于 进程空间的 内核空间，可进行进程间 & 进程内交互

所以，原理图可表示为以下：

> 虚线表示并非直接交互

![示意图](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329095718.png)



#### 说明2： `Binder`驱动 & `Service Manager`进程 属于 `Android`基础架构（即系统已经实现好了）；而`Client` 进程 和 `Server` 进程 属于`Android`应用层（需要开发者自己实现）

![示意图](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329095746.png)



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210329170322)



#### 说明3：Binder请求的线程管理

* Server进程会创建很多线程来处理Binder请求
* Binder模型的线程管理 采用Binder驱动的线程池，并由Binder驱动自身进行管理
  而不是由Server进程来管理的

* 一个进程的Binder线程数默认最大是16，超过的请求会被阻塞等待空闲的Binder线程。
  所以，在进程间通信时处理并发问题时，如使用ContentProvider时，它的CRUD（创建、检索、更新和删除）方法只能同时有16个线程同时工作

* 至此，我相信大家对Binder 跨进程通信机制 模型 已经有了一个非常清晰的定性认识
* 下面，我将通过一个实例，分析Binder跨进程通信机制 模型在 Android中的具体代码实现方式
  即分析 上述步骤在Android中具体是用代码如何实现的


