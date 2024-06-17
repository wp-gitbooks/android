# 参考
http://gityuan.com/2017/04/02/android-application/


# 概述
system进程和app进程都运行着一个或多个app，每个app都会有一个对应的Application对象(该对象 跟LoadedApk一一对应)。下面分别以下两种进程创建Application的过程：

- system_server进程；
- app进程


## 顺序
Application.attachBaseContext(super before) -> Application.attachBaseContext(super after) -> ContentProvider.attachInfo(super before) -> ContentProvider.onCreate() -> ContentProvider.attachInfo(super after) -> Application.onCreate(super before) -> Application.onCreate(super after)



# 总结
## (一)system_server进程
其application创建过程都创建对象有ActivityThread，Instrumentation, ContextImpl，LoadedApk，Application。 流程图如下：

![system_application](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210425143712.jpg)

## (二) app进程
其application创建过程都创建对象有ActivityThread，ContextImpl，LoadedApk，Application。 流程图如下：

![app_application](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210425143806.jpg)
App进程的Application创建过程，跟system进程的核心逻辑都差不多。只是app进程多了两次binder调用。


# 面试题
## SystemServer相关

1.SystemServer是如何被fork出来的 

2.SystemServer做了些什么事情 

3.SystemServiceManager是负责什么的？  

4.SystemServiceManager是如何创建的？ 

5.ServiceManager是负责什么的？ 

6.启动服务有几种方式？他们之间有什么区别？ 

7.SystemServiceManager和ServiceManager有什么区别？ 

8.LocalService是负责什么工作的？ 

9.SystemServer是服务端进程，那么谁是客户端进程？他们是如何通信的？内部机制是什么？（Binder相关的问题）