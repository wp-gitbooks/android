# 线索
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303031008534.png)

linux-------->进程----------->IPC

![image-20210524185027602](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210524185027.png)



![image-20210524185138526](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210524185138.png)



# Binder

## 是什么？用途？  
Binder是Android系统中进程间通讯（IPC）的一种方式，也是Android系统中最重要的特性之一。Android中的四大组件Activity，Service，Broadcast，ContentProvider，不同的App等都运行在不同的进程中，它是这些进程间通讯的桥梁。正如其名“粘合剂”一样，它把系统中各个组件粘合到了一起，是各个组件的桥梁。
![定义](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210329094328.png)
现在我们可以对 Binder 做个更加全面的定义了：
-   从进程间通信的角度看，Binder 是一种进程间通信的机制；
-   从 Server 进程的角度看，Binder 指的是 Server 中的 Binder 实体对象；
-   从 Client 进程的角度看，Binder 指的是对 Binder 代理对象，是 Binder 实体对象的一个远程代理
-   从传输过程的角度看，Binder 是一个可以跨进程传输的对象；Binder 驱动会对这个跨越进程边界的对象对一点点特殊处理，自动完成代理对象和本地对象之间的转换
  
### Binder到底是什么？

我们经常提到Binder，那么Binder到底是什么呢？

Binder的设计采用了面向对象的思想，在Binder通信模型的四个角色里面；他们的代表都是“Binder”，这样，对于Binder通信的使用者而言，Server里面的Binder和Client里面的Binder没有什么不同，一个Binder对象就代表了所有，它不用关心实现的细节，甚至不用关心驱动以及SM的存在；这就是抽象。

-   通常意义下，Binder指的是一种通信机制；我们说AIDL使用Binder进行通信，指的就是**Binder这种IPC机制**。
  
-   对于Server进程来说，Binder指的是**Binder本地对象**
  
-   对于Client来说，Binder指的是**Binder代理对象**，它只是**Binder本地对象**的一个远程代理；对这个Binder代理对象的操作，会通过驱动最终转发到Binder本地对象上去完成；对于一个拥有Binder对象的使用者而言，它无须关心这是一个Binder代理对象还是Binder本地对象；对于代理对象的操作和对本地对象的操作对它来说没有区别。
  
-   对于传输过程而言，Binder是可以进行跨进程传递的对象；Binder驱动会对具有跨进程传递能力的对象做特殊处理：自动完成代理对象和本地对象的转换。
  

> 面向对象思想的引入将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法，而其独特之处在于Binder对象是一个可以跨进程引用的对象，它的实体（本地对象）位于一个进程中，而它的引用（代理对象）却遍布于系统的各个进程之中。最诱人的是，这个引用和java里引用一样既可以是强类型，也可以是弱类型，而且可以从一个进程传给其它进程，让大家都能访问同一Server，就象将一个对象或引用赋值给另一个引用一样。Binder模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行于同一个面向对象的程序之中。形形色色的Binder对象以及星罗棋布的引用仿佛粘接各个应用程序的胶水，这也是Binder在英文里的原意

### 驱动里面的Binder 
Binder驱动是Android专用的，但底层的驱动架构与Linux驱动一样。binder驱动在以misc设备进行注册，作为虚拟字符设备，没有直接操作硬件，只是对设备内存的处理。主要是驱动设备的初始化(binder_init)，打开 (binder_open)，映射(binder_mmap)，数据操作(binder_ioctl)。

![binder_driver](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210309222632.png)



我们现在知道，Server进程里面的Binder对象指的是Binder本地对象，Client里面的对象值得是Binder代理对象；在Binder对象进行跨进程传递的时候，Binder驱动会自动完成这两种类型的转换；因此Binder驱动必然保存了每一个跨越进程的Binder对象的相关信息；在驱动中，Binder本地对象的代表是一个叫做`binder_node`的数据结构，Binder代理对象是用`binder_ref`代表的；有的地方把Binder本地对象直接称作Binder实体，把Binder代理对象直接称作Binder引用（句柄），其实指的是Binder对象在驱动里面的表现形式；读者明白意思即可。

OK，现在大致了解Binder的通信模型，也了解了Binder这个对象在通信过程中各个组件里面到底表示的是什么

### 深入理解Java层的Binder

![image-20210407155008863](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210407155009.png)

aidl接口

// ICompute.aidl

```java
package com.example.test.app;

interface ICompute {

 int add(int a, int b);

}
```

编译后：
```java
/*

 * This file is auto-generated.  DO NOT MODIFY.

 */

package com.analysys.analysystrack;

public interface ICompute extends android.os.IInterface

{

 /** Default implementation for ICompute. */

 public static class Default implements com.analysys.analysystrack.ICompute

 {

 @Override public int add(int a, int b) throws android.os.RemoteException

 {

 return 0;

 }

 @Override

 public android.os.IBinder asBinder() {

 return null;

 }

 }

 /** Local-side IPC implementation stub class. */

 public static abstract class Stub extends android.os.Binder implements com.analysys.analysystrack.ICompute

 {

 private static final java.lang.String DESCRIPTOR = "com.analysys.analysystrack.ICompute";

 /** Construct the stub at attach it to the interface. */

 public Stub()

 {

 this.attachInterface(this, DESCRIPTOR);

 }

 /**

 * Cast an IBinder object into an com.analysys.analysystrack.ICompute interface,

 * generating a proxy if needed.

 */

 public static com.analysys.analysystrack.ICompute asInterface(android.os.IBinder obj)

 {

 if ((obj==null)) {

 return null;

 }

 android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);

 if (((iin!=null)&&(iin instanceof com.analysys.analysystrack.ICompute))) {

 return ((com.analysys.analysystrack.ICompute)iin);

 }

 return new com.analysys.analysystrack.ICompute.Stub.Proxy(obj);

 }

 @Override public android.os.IBinder asBinder()

 {

 return this;

 }

 @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException

 {

 java.lang.String descriptor = DESCRIPTOR;

 switch (code)

 {

 case INTERFACE_TRANSACTION:

 {

 reply.writeString(descriptor);

 return true;

 }

 case TRANSACTION_add:

 {

 data.enforceInterface(descriptor);

 int _arg0;

 _arg0 = data.readInt();

 int _arg1;

 _arg1 = data.readInt();

 int _result = this.add(_arg0, _arg1);

 reply.writeNoException();

 reply.writeInt(_result);

 return true;

 }

 default:

 {

 return super.onTransact(code, data, reply, flags);

 }

 }

 }

 private static class Proxy implements com.analysys.analysystrack.ICompute

 {

 private android.os.IBinder mRemote;

 Proxy(android.os.IBinder remote)

 {

 mRemote = remote;

 }

 @Override public android.os.IBinder asBinder()

 {

 return mRemote;

 }

 public java.lang.String getInterfaceDescriptor()

 {

 return DESCRIPTOR;

 }

 @Override public int add(int a, int b) throws android.os.RemoteException

 {

 android.os.Parcel _data = android.os.Parcel.obtain();

 android.os.Parcel _reply = android.os.Parcel.obtain();

 int _result;

 try {

 _data.writeInterfaceToken(DESCRIPTOR);

 _data.writeInt(a);

 _data.writeInt(b);

 boolean _status = mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);

 if (!_status && getDefaultImpl() != null) {

 return getDefaultImpl().add(a, b);

 }

 _reply.readException();

 _result = _reply.readInt();

 }

 finally {

 _reply.recycle();

 _data.recycle();

 }

 return _result;

 }

 public static com.analysys.analysystrack.ICompute sDefaultImpl;

 }

 static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);

 public static boolean setDefaultImpl(com.analysys.analysystrack.ICompute impl) {

 if (Stub.Proxy.sDefaultImpl == null && impl != null) {

 Stub.Proxy.sDefaultImpl = impl;

 return true;

 }

 return false;

 }

 public static com.analysys.analysystrack.ICompute getDefaultImpl() {

 return Stub.Proxy.sDefaultImpl;

 }

 }

 public int add(int a, int b) throws android.os.RemoteException;

}

​

asInterface

Proxy

Service:

package com.analysys.analysystrack;

​

import android.app.Service;

import android.content.Intent;

import android.os.IBinder;

import android.os.RemoteException;

​

class LocalService extends Service {

​

 ICompute.Stub stub = new ICompute.Stub() {

 @Override

 public int add(int a, int b) throws RemoteException {

 return a+b;

 }

 };

​

 @Override

 public IBinder onBind(Intent intent) {

 return stub;

 }

}

```



Client:

```java
Intent intent = new Intent();

 ServiceConnection conn = new ServiceConnection() {

 @Override

 public void onServiceConnected(ComponentName componentName, IBinder iBinder) {

 ICompute iCompute = ICompute.Stub.asInterface(iBinder);

 try {

 iCompute.add(1,1);

 } catch (RemoteException e) {

 e.printStackTrace();

 }

 }



 @Override

 public void onServiceDisconnected(ComponentName componentName) {



 }

 };



 bindService(intent,conn, Context.BIND_AUTO_CREATE);


```


## 为什么用？
![image-20210708191239800](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708191239.png)

[https://blog.csdn.net/universus/article/details/6211589](https://blog.csdn.net/universus/article/details/6211589)

Android 系统是基于 Linux 内核的，Linux 已经提供了管道、消息队列、共享内存和 Socket 等 IPC 机制。那为什么 Android 还要提供 Binder 来实现 IPC 呢？主要是基于**性能**、**稳定性**和**安全性**几方面的原因。


### Binder机制同其他进程机制对比

对比 `Linux` （`Android`基于`Linux`）上的其他进程通信方式（管道、消息队列、共享内存、 信号量、`Socket`），`Binder` 机制的优点有：

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210329104527.png)

[https://blog.csdn.net/universus/article/details/6211589](https://blog.csdn.net/universus/article/details/6211589)

Android 系统是基于 Linux 内核的，Linux 已经提供了管道、消息队列、共享内存和 Socket 等 IPC 机制。那为什么 Android 还要提供 Binder 来实现 IPC 呢？主要是基于**性能**、**稳定性**和**安全性**几方面的原因。

#### 性能

首先说说性能上的优势。Socket 作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。消息队列和管道采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用。Binder 只需要一次数据拷贝，性能上仅次于共享内存。

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210325154449.jpg)

#### 稳定性

再说说稳定性，Binder 基于 C/S 架构，客户端（Client）有什么需求就丢给服务端（Server）去完成，**架构清晰、职责明确又相互独立**，自然稳定性更好。共享内存虽然无需拷贝，但实现方式复杂，没有客户与服务端之别， 需要充分考虑到**访问临界资源的并发同步问题**，否则可能会出现死锁等问题。从稳定性的角度讲，Binder 机制是优于内存共享的。

#### 安全性

https://blog.csdn.net/haoxuhong/article/details/80140911?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.baidujs&dist_request_id=&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.baidujs

https://blog.csdn.net/javac_jj/article/details/94388255

另一方面就是安全性。Android 作为一个开放性的平台，市场上有各类海量的应用供用户选择安装，因此安全性对于 Android 平台而言极其重要。作为用户当然不希望我们下载的 APP 偷偷读取我的通信录，上传我的隐私数据，后台偷跑流量、消耗手机电量。传统的 IPC 没有任何安全措施，完全依赖上层协议来确保。首先**传统的 IPC 接收方无法获得对方可靠的进程用户ID/进程ID（UID/PID）**，从而无法鉴别对方身份。Android 为每个安装好的 **APP 分配了自己的 UID**，故而进程的 UID 是鉴别进程身份的重要标志。传统的 IPC 只能由用户在数据包中填入 UID/PID，但这样不可靠，容易被恶意程序利用。可靠的身份标识只有由 IPC 机制在内核中添加。其次传统的 IPC 访问接入点是开放的，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法阻止恶意程序通过猜测接收方地址获得连接。同时 Binder 既支持实名 Binder，又支持匿名 Binder，安全性高。

### 总结 Binder 的优势

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210325154659.jpg)

## IPC原理  [[0-进程#Binder IPC原理]]
### mmap [[Android匿名共享内存]] [[2-mmap内存映射]]

![binder_physical_memory](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210309223135.jpg)


![binder_memory_map](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519114651.jpg)


#### mmap 应用 [[mmap应用-京东移动日志]]



## Binder机制跨进程原理
上文给出了Binder的通信模型，指出了通信过程的四个角色: Client, Server, SM, driver; 但是我们仍然不清楚**Client到底是如何与Server完成通信的**。

两个运行在用户空间的进程A和进程B如何完成通信呢？内核可以访问A和B的所有数据；所以，最简单的方式是通过内核做中转；假设进程A要给进程B发送数据，那么就先把A的数据copy到内核空间，然后把内核空间对应的数据copy到B就完成了；用户空间要操作内核空间，需要通过系统调用；刚好，这里就有两个系统调用：`copy_from_user`, `copy_to_user`。

但是，Binder机制并不是这么干的。讲这么一段，是说明进程间通信并不是什么神秘的东西。那么，Binder机制是如何实现跨进程通信的呢？

Binder驱动为我们做了一切。

假设Client进程想要调用Server进程的`object`对象的一个方法`add`;对于这个跨进程通信过程，我们来看看Binder机制是如何做的。 （通信是一个广泛的概念，只要一个进程能调用另外一个进程里面某对象的方法，那么具体要完成什么通信内容就很容易了。）

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210729144404.png)

首先，Server进程要向SM注册；告诉自己是谁，自己有什么能力；在这个场景就是Server告诉SM，它叫`zhangsan`，它有一个`object`对象，可以执行`add` 操作；于是SM建立了一张表：`zhangsan`这个名字对应进程Server;

然后Client向SM查询：我需要联系一个名字叫做`zhangsan`的进程里面的`object`对象；这时候关键来了：进程之间通信的数据都会经过运行在内核空间里面的驱动，驱动在数据流过的时候做了一点手脚，它并不会给Client进程返回一个真正的`object`对象，而是返回一个看起来跟`object`一模一样的代理对象`objectProxy`，这个`objectProxy`也有一个`add`方法，但是这个`add`方法没有Server进程里面`object`对象的`add`方法那个能力；`objectProxy`的`add`只是一个傀儡，它唯一做的事情就是把参数包装然后交给驱动。(这里我们简化了SM的流程，见下文)

但是Client进程并不知道驱动返回给它的对象动过手脚，毕竟伪装的太像了，如假包换。Client开开心心地拿着`objectProxy`对象然后调用`add`方法；我们说过，这个`add`什么也不做，直接把参数做一些包装然后直接转发给Binder驱动。

驱动收到这个消息，发现是这个`objectProxy`；一查表就明白了：我之前用`objectProxy`替换了`object`发送给Client了，它真正应该要访问的是`object`对象的`add`方法；于是Binder驱动通知Server进程，_调用你的object对象的`add`方法，然后把结果发给我_，Sever进程收到这个消息，照做之后将结果返回驱动，驱动然后把结果返回给`Client`进程；于是整个过程就完成了。

由于驱动返回的`objectProxy`与Server进程里面原始的`object`是如此相似，给人感觉好像是**直接把Server进程里面的对象object传递到了Client进程**；因此，我们可以说**Binder对象是可以进行跨进程传递的对象**

但事实上我们知道，Binder跨进程传输并不是真的把一个对象传输到了另外一个进程；传输过程好像是Binder跨进程穿越的时候，它在一个进程留下了一个真身，在另外一个进程幻化出一个影子（这个影子可以很多个）；Client进程的操作其实是对于影子的操作，影子利用Binder驱动最终让真身完成操作。

理解这一点非常重要；务必仔细体会。另外，Android系统实现这种机制使用的是_代理模式_, 对于Binder的访问，如果是在同一个进程（不需要跨进程），那么直接返回原始的Binder实体；如果在不同进程，那么就给他一个代理对象（影子）；我们在系统源码以及AIDL的生成代码里面可以看到很多这种实现。

另外我们为了简化整个流程，隐藏了SM这一部分驱动进行的操作；实际上，由于SM与Server通常不在一个进程，Server进程向SM注册的过程也是跨进程通信，驱动也会对这个过程进行暗箱操作：SM中存在的Server端的对象实际上也是代理对象，后面Client向SM查询的时候，驱动会给Client返回另外一个代理对象。Sever进程的本地对象仅有一个，其他进程所拥有的全部都是它的代理。

一句话总结就是：**Client进程只不过是持有了Server端的代理；代理对象协助驱动完成了跨进程通信。**


## 通信模型
一次完成的进程间通信必然至少包含两个进程, 通常我们称通信的双方分别为客户端进程(Client) 和服务端进程(Server), 由于进程隔离机制的存在, 通信双方必然需要借助 Binder 来实现.

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528143615.jpeg)



![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519154235.png)



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519154331.png)


### Client/Server/ServiceManager/Binder驱动
BInder 是基于 C/S 架构. 是由一些列组件组成. 包括 Client, Server, ServiceManager, Binder 驱动.

> -   Client, Server, ServiceManager 运行在用户空间
>     
> -   Binder 驱动运行在内核空间.
>     

> -   ServiceManager, Binder 驱动由系统提供.
>     
> -   Client, Service 由应用程序来实现
>     

> Client, Server, ServiceManager 都是通过系统调用 open, mmap,和ioctl 来访问设备文件 /dev/binder, 从而实现与 Binder 驱动的交互来间接的实现跨进程通信.

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202304130921069.png)



![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210720180542.png)


### Binder 驱动

Binder 驱动就如如同路由器一样, 是整个通信的核心. 驱动负责进程之间 Binder 通信的建立 / 传递, Binder 引用计数管理, 数据包在进程之间的传递和交互等一系列底层支持.

### ServiceManager 与 实名 Binder

ServiceManager 作用是将字符形式的 Binder 名字转化成 Client 中对该 Binder 的引用, 使得 Client 能够通过 Binder 的名字获得对 Binder 实体的引用.

注册了名字的 Binder 叫实名 Binder, 就像网站一样除了 IP 地址以外还有自己的网址. Server 创建了 Binder, 并为它起一个字符形式, 可读易记的名字, 将这个 BInder 实体连同名字一起以数据包的形式通过 Binder 驱动 发送给 ServiceManager, 通知 ServiceManager 注册一个名字为 "张三"的 Binder, 它位于某个 Server 中, 驱动为这个穿越进程边界的 BInder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用, 将名字以及新建的引用打包传给 ServiceManager, ServiceManager 收到数据后从中取出名字和引用填入查找表.

ServiceManager 是一个进程, Server 又是一个另外的进程, Server 向



### Binder 进程与线程 [[Binder线程池工作原理]]
![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528143951.jpeg)

对于底层Binder驱动，通过 binder_procs 链表记录所有创建的 binder_proc 结构体，binder 驱动层的每一个 binder_proc 结构体都与用户空间的一个用于 binder 通信的进程一一对应，且每个进程有且只有一个 ProcessState 对象，这是通过单例模式来保证的。在每个进程中可以有很多个线程，每个线程对应一个 IPCThreadState 对象，IPCThreadState 对象也是单例模式，即一个线程对应一个 IPCThreadState 对象，在 Binder 驱动层也有与之相对应的结构，那就是 Binder_thread 结构体。在 binder_proc 结构体中通过成员变量 rb_root threads，来记录当前进程内所有的 binder_thread。

**Binder 线程池：** 每个 Server 进程在启动时创建一个 binder 线程池，并向其中注册一个 Binder 线程；之后 Server 进程也可以向 binder 线程池注册新的线程，或者 Binder 驱动在探测到没有空闲 binder 线程时主动向 Server 进程注册新的的 binder 线程。对于一个 Server 进程有一个最大 Binder 线程数限制，默认为16个 binder 线程，例如 Android 的 _server 进程就存在16个线程。对于所有 Client 端进程的 binder 请求都是交由 Server 端进程的 binder 线程来处理的

#### ProcessState::self()主要工作

-   调用open()，打开/dev/binder驱动设备；
  
-   再利用mmap()，创建大小为`1M-8K`的内存地址空间；
  
-   设定当前进程最大的最大并发Binder线程个数为`16`



#### 线程池数量
默认最大Binder线程数量：15个 (通过ProcessState.cpp中看到DEFAULT_MAX_BINDER_THREADS）


# 核心组成 [[#通信模型]]
## Binder驱动

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528143635.jpeg)

### 流程

![binder_driver](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519114333.png)

### 系统调用
![binder_syscall](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519114351.png)


## ServerManager [[ServiceManager]]


## mmap 
[[2-mmap内存映射]]
[[mmap-是什么-为什么-怎么用]]
[[Android匿名共享内存]]


### mmap原理

![image-20210708161125465](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708161125.png)

![image-20210708161143411](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708161143.png)

![image-20210708161206623](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708161206.png)





### Binder是如何做到一次拷贝？
内核空间的**虚拟内存**和接收方的**用户空间**的虚拟内存同时映射在内核地址空间

![image-20210708161102914](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708161102.png)


### 为什么Binder中客户端不对内存进行映设？
（如果映设就是内存共享mmap机制，这样就要对比Binder和mmap的区别）
![image-20210708161353238](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708161353.png)

![image-20210708161433760](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708161433.png)



### 数据大小 [[一次Binder通信最大可以传输多大的数据]]
Binder通信最大可传输多大数据 ?

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210701061802.png)

默认多大：1M-8K (通过**ProcessState**.cpp中看到BINDER_VM_SIZE)


#### intent传递的最大数据
startActivity 一般都是 oneway机制，最大 (1M-8K)/2

数据传输过大解决方案：
一. 限制传递数据量

二. 改变数据传输方式 [[Activity之间传递数据的方式及常见问题总结]]
1. 静态static

2. 单例

3. Application

4. 持久化


# 通信协议

## 概述

Binder 是 Android 中的 IPC（进程间通信）的最要一环，它的作用就是：

1、异步调用（单个binder请求）
应用向 binder 驱动发送数据后不需要挂起线程等待 binder 驱动的回复，而是直接结束。



2、串行化处理（多个binder请求）
对于一个服务端的 AIDL 接口而言，所有的 oneway 方法不会同时执行，binder 驱动会将他们串行化处理，排队一个一个调用。
像一些系统服务调用应用进程的时候就会使用 oneway，比如 **AMS 调用应用进程启动 Activity**，这样就算应用进程中做了耗时的任务，也不会阻塞系统服务的运行。



### 阻塞与否对比  [[1-如何使用AIDL#in、out和inout]]

oneway：调用方非阻塞（non-block）
非 oneway：调用方阻塞（休眠）

#### 非oneway机制

![binder_protocol](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519114511.jpg)



#### oneway机制

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528140445.png)



从通信协议的角度来看这个过程:

![binder_transaction](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210729154200.jpg)

- Binder客户端或者服务端向Binder Driver发送的命令都是以BC_开头,例如本文的`BC_TRANSACTION`和`BC_REPLY`, 所有Binder Driver向Binder客户端或者服务端发送的命令则都是以BR_开头, 例如本文中的`BR_TRANSACTION`和`BR_REPLY`.
- 只有当`BC_TRANSACTION`或者`BC_REPLY`时, 才调用binder_transaction()来处理事务. 并且都会回应调用者一个`BINDER_WORK_TRANSACTION_COMPLETE`事务, 经过binder_thread_read()会转变成`BR_TRANSACTION_COMPLETE`.
- startService过程便是一个非oneway的过程, 那么oneway的通信过程如下所述.

## 说一说oneway

上图是非oneway通信过程的协议图, 下图则是对于oneway场景下的通信协议图:

![binder_transaction_oneway](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210729154218.jpg)

当收到BR_TRANSACTION_COMPLETE则程序返回,有人可能觉得好奇,为何oneway怎么还要等待回应消息? 我举个例子,你就明白了.

你(app进程)要给远方的家人(system_server进程)邮寄一封信(transaction), 你需要通过邮寄员(Binder Driver)来完成.整个过程如下:

1. 你把信交给邮寄员(`BC_TRANSACTION`);
2. 邮寄员收到信后, 填一张单子给你作为一份回执(`BR_TRANSACTION_COMPLETE`). 这样你才放心知道邮递员已确定接收信, 否则就这样走了,信到底有没有交到邮递员手里都不知道,这样的通信实在太让人不省心, 长时间收不到远方家人的回信, 无法得知是在路的中途信件丢失呢,还是压根就没有交到邮递员的手里. 所以说oneway也得知道信是投递状态是否成功.
3. 邮递员利用交通工具(Binder Driver),将信交给了你的家人(`BR_TRANSACTION`);

当你收到回执(BR_TRANSACTION_COMPLETE)时心里也不期待家人回信, 那么这便是一次oneway的通信过程.

如果你希望家人回信, 那便是非oneway的过程,在上述步骤2后并不是直接返回,而是继续等待着收到家人的回信, 经历前3个步骤之后继续执行:

1. 家人收到信后, 立马写了个回信交给邮递员`BC_REPLY`;
2. 同样,邮递员要写一个回执(`BR_TRANSACTION_COMPLETE`)给你家人;
3. 邮递员再次利用交通工具(Binder Driver), 将回信成功交到你的手上(`BR_REPLY`)

这便是一次完成的非oneway通信过程.

oneway与非oneway: 都是需要等待Binder Driver的回应消息BR_TRANSACTION_COMPLETE. 主要区别在于oneway的通信收到BR_TRANSACTION_COMPLETE则返回,而不会再等待BR_REPLY消息的到来. 另外，oneway的binder IPC则接收端无法获取对方的pid.









# Binder四层源码分析
## 一次完整通信

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528144609.jpeg)

-   我们在使用 Binder 时基本都是调用 framework 层封装好的方法，AIDL 就是 framework 层提供的傻瓜式是使用方式。假设服务已经注册完，我们来看看客户端怎么执行服务端的方法。
  
-   首先我们通过 ServiceManager 获取到服务端的 BinderProxy 代理对象，通过调用 BinderProxy 将参数，方法标识（例如：TRANSACTION_test，AIDL中自动生成）传给 ServiceManager，同时客户端线程进入等待状态。
  
-   ServiceManager 将用户空间的参数等请求数据复制到内核空间，并向服务端插入一条执行执行方法的事务。事务执行完通知 ServiceManager 将执行结果从内核空间复制到用户空间，并唤醒等待的线程，响应结果，通讯结束


## Binder native层

## Binder驱动

## Binder jni层

## Binder Framework层实现



# Binder 架构
http://gityuan.com/2015/11/21/binder-framework/



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210528143541.jpeg)



- Binder 通信采用 C/S 架构，从组件视角来说，包含 Client、 Server、 ServiceManager 以及 Binder 驱动，其中 ServiceManager 用于管理系统中的各种服务。
- Binder 在 framework 层进行了封装，通过 JNI 技术调用 Native（C/C++）层的 Binder 架构。
- Binder 在 Native 层以 ioctl 的方式与 Binder 驱动通讯



## framework

## 架构图

![java_binder](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519142415.jpg)



## 类图

![class_java_binder](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519142440.jpg)


-   **ServiceManager：**通过getIServiceManager方法获取的是ServiceManagerProxy对象； ServiceManager的addService, getService实际工作都交由ServiceManagerProxy的相应方法来处理；
  
-   **ServiceManagerProxy：**其成员变量mRemote指向BinderProxy对象，ServiceManagerProxy的addService, getService方法最终是交由mRemote来完成。
  
-   **ServiceManagerNative**：其方法asInterface()返回的是ServiceManagerProxy对象，ServiceManager便是借助ServiceManagerNative类来找到ServiceManagerProxy；
  
-   **Binder：**其成员变量mObject和方法execTransact()用于native方法
  
-   **BinderInternal：**内部有一个GcWatcher类，用于处理和调试与Binder相关的垃圾回收。
  
-   **IBinder：**接口中常量FLAG_ONEWAY：客户端利用binder跟服务端通信是阻塞式的，但如果设置了FLAG_ONEWAY，这成为非阻塞的调用方式，客户端能立即返回，服务端采用回调方式来通知客户端完成情况。另外IBinder接口有一个内部接口DeathDecipient(死亡通告)



## 类分层

![java_binder_framework](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210519142502.jpg)


# 借鉴地方-设计巧妙-设计模式？


# 应用 [[Binder应用]]
## AIDL [[1-如何使用AIDL]]
http://gityuan.com/2015/11/23/binder-aidl/



## AMS、PMS

## 手写Binder一次拷贝机制，抛弃ShareFreferences存储
### 如何使用Binder

[http://gityuan.com/2015/11/22/binder-use/](http://gityuan.com/2015/11/22/binder-use/)

[http://weishu.me/2016/01/12/binder-index-for-newer/](http://weishu.me/2016/01/12/binder-index-for-newer/)

## mmkv


# Binder权限控制
[[Binder-IPC的权限控制]]

## Binder权限校验使用
[[Binder权限校验使用]]


# 面试题
![image-20210628111602820](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210628111602.png)


## 为什么Android要采用binder作为IPC机制


## binder机制是如何跨进程的


## 一次拷贝是发生在客户端还是服务端


## 为什么activity间传递对象需要序列化


# 参考
http://weishu.me/2016/01/12/binder-index-for-newer/
http://gityuan.com/2015/11/22/binder-use/
https://blog.csdn.net/universus/article/details/6211589