# Service和Thread的区别？

> Service有两种存在方式，一种是在另外一个进程的**主线程中**，一种是在当前进程的主线程中。
>  Thread只能以某一个进程中**非主线程**的方式存在。

## 详情

servie是系统的组件，它由系统进程托管（servicemanager），它们之间的通信类似于client和server，是一种轻量级的ipc通信，这种通信的载体是binder，它是在linux层交换信息的一种ipc。而thread是由本应用程序托管。ipc是linux中的一种进程间通信方式。

## 深入一步

> 1). Thread：Thread 是程序执行的最小单元，它是分配CPU的基本单位。可以用 Thread 来执行一些异步的操作。

> 2). Service：Service 是android的一种机制，如果不设置process为运行在主进程中。
>  在AndroidManifest中设置类似于android:process=":remote"会独立进程，冒号后面的名字由自己决定。
>  新进程的名字就会是<package>:remote(类似于com.github.example:remote)

## 为什么使用Service,不用Thread

> 既然这样，那么我们为什么要用 Service 呢？其实这跟 android 的系统机制有关，我们先拿 Thread 来说。Thread 的运行是独立于 Activity 的，也就是说当一个 Activity 被 finish 之后，如果你没有主动停止 Thread 或者 Thread 里的 run 方法没有执行完毕的话，Thread 也会一直执行。因此这里会出现一个问题：当 Activity 被 finish 之后，你不再持有该 Thread 的引用。另一方面，你没有办法在不同的 Activity 中对同一 Thread 进行控制。

> 举个例子：如果你的 Thread 需要不停地隔一段时间就要连接服务器做某种同步的话，该 Thread 需要>在 Activity 没有start的时候也在运行。这个时候当你 start 一个 Activity 就没有办法在该 Activity 里面控制之前创建的 Thread。因此你便需要创建并启动一个 Service ，在 Service 里面创建、运行并控制该 Thread，这样便解决了该问题（因为任何 Activity 都可以控制同一 Service，而系统也只会创建一个对应 Service 的实例）。

> 因此你可以把 Service 想象成一种消息服务，而你可以在任何有 Context 的地方调用 Context.startService、Context.stopService、Context.bindService，Context.unbindService，来控制它，你也可以在 Service 里注册 BroadcastReceiver，在其他地方通过发送 broadcast 来控制它，当然这些都是 Thread 做不到的。

## 关于Service的存活

> 根据进程优先级，Thread在后台运行（Activty stop）的优先级低于后台运行的Service，如果执行系统资源紧张，会优先杀死前一种，后台运行的Service一般情况下不会被杀死，如果被杀死，系统空闲时会重新启动service.

> 可以设置 startCommand的返回值
>  return START_STICKY;（粘性的service，内存不足杀死，内存足够启动）
>  return START_NOT_STICKY（非粘性的service，应用关闭，直接关闭）

## 关于不死Service

> 设置了START_STICKY的service其实已经初步具有不死的属性，双线程守护(java)在小米7.0不起作用。应该是有白名单之类的东西的关系。ndk没有测试。在8.0虚拟机好像能起到不死作用，但是没有打印,,,不是很确定。





# 区别

Service

1.在**后台用来操作时间跨度较长**的工作应用组件。

2.Service的生命周期方法在主线程中执行，如果想执行一个长时间的工作，需要开启一个分线程（Thread）。

3.应用退出，Service不会停止。

4.**再次启动应用时，还可以以正在运行的Service通信**。



Thread

1.用来开启一个**分线程的类，做一个长时间的工作**。

2.Thread类的run( )在分线程中执行。

3.应用退出，Thread也不会停止。

4.**再次启动应用，不能再控制之前的Thread对象**。



# 参考

https://www.jianshu.com/p/bde28b81a81c