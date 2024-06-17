# Application创建过程

http://gityuan.com/2017/04/02/android-application/





# Activity

## Activity启动流程图



![start_activity_process](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210310220039.jpg)



![start_activity](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210310220109.jpg)





# service

## startService启动过程分析

http://gityuan.com/2016/03/06/start-service/

![start_service](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210310220427.png)





![start_service_process](http://gityuan.com/images/android-service/start_service/start_service_processes.jpg)

## bindService启动过程分析

http://gityuan.com/2016/05/01/bind-service/



## unbindService流程分析

http://gityuan.com/2016/05/02/unbind-service/



## 以Binder视角来看Service启动

http://gityuan.com/2016/09/04/binder-start-service/



# Broadcast广播机制分析

http://gityuan.com/2016/06/04/broadcast-receiver/



# ContentProvider原理

http://gityuan.com/2016/07/30/content-provider/

http://gityuan.com/2016/05/03/content_provider_release/



# 四大组件之综述

## 四大组件的启动

http://gityuan.com/2016/10/09/app-process-create-2/



http://gityuan.com/2017/05/19/ams-abstract/





# 四大组件之Record

## 四大组件之ActivityRecord

http://gityuan.com/2017/06/11/activity_record/

## 四大组件之ServiceRecord

http://gityuan.com/2017/05/25/service_record/

## 四大组件之BroadcastRecord

http://gityuan.com/2017/06/03/broadcast_record/



## 四大组件之ContentProviderRecord

http://gityuan.com/2017/06/04/content_provider_record/



# Android Context

http://gityuan.com/2017/04/09/android_context/

https://zhuanlan.zhihu.com/p/222466615



# AMS总结(一)

http://gityuan.com/2017/06/25/ams_summary_1/



# 面试题

## Activity



### Activity的启动过程（不要回答生命周期）

pp启动的过程有两种情况，第一种是从桌面launcher上点击相应的应用图标，第二种是在activity中通过调用startActivity来启动一个新的activity。

我们创建一个新的项目，默认的根activity都是MainActivity，而所有的activity都是保存在堆栈中的，我们启动一个新的activity就会放在上一个activity上面，而我们从桌面点击应用图标的时候，由于launcher本身也是一个应用，当我们点击图标的时候，系统就会调用startActivitySately(),一般情况下，我们所启动的activity的相关信息都会保存在intent中，比如action，category等等。

我们在安装这个应用的时候，系统也会启动一个PackaManagerService的管理服务，这个管理服务会对AndroidManifest.xml文件进行解析，从而得到应用程序中的相关信息，比如service，activity，Broadcast等等，然后获得相关组件的信息。当我们点击应用图标的时候，就会调用startActivitySately()方法，而这个方法内部则是调用startActivty(),而startActivity()方法最终还是会调用startActivityForResult()这个方法。而在startActivityForResult()这个方法。因为startActivityForResult()方法是有返回结果的，所以系统就直接给一个-1，就表示不需要结果返回了。

而startActivityForResult()这个方法实际是通过Instrumentation类中的execStartActivity()方法来启动activity，Instrumentation这个类主要作用就是监控程序和系统之间的交互。而在这个execStartActivity()方法中会获取ActivityManagerService的代理对象，通过这个代理对象进行启动activity。启动会就会调用一个checkStartActivityResult()方法，如果说没有在配置清单中配置有这个组件，就会在这个方法中抛出异常了。

当然最后是调用的是Application.scheduleLaunchActivity()进行启动activity，而这个方法中通过获取得到一个ActivityClientRecord对象，而这个ActivityClientRecord通过handler来进行消息的发送，系统内部会将每一个activity组件使用ActivityClientRecord对象来进行描述，而ActivityClientRecord对象中保存有一个LoaderApk对象，通过这个对象调用handleLaunchActivity来启动activity组件，而页面的生命周期方法也就是在这个方法中进行调用。





### 如果需要在Activity间传递大量的数据怎么办？



### 打开多个页面，如何实现一键退出?



### 四大组件的生命周期和简单用法



### activity启动模式，有哪些不同



### Activity生命周期？



### 四大组件，fileprovider和Contentprovide区别，activity启动流程



### 启动未注册的Activity



## 



### 保存Activity状态

onSaveInstanceState(Bundle)会在activity转入后台状态之前被调用，也就是onStop()方法之前，onPause方法之后被调用；

### 四种LaunchMode及其使用场景

此处延伸：栈(First In Last Out)与队列(First In First Out)的区别

栈与队列的区别：

\1. 队列先进先出，栈先进后出

\2. 对插入和删除操作的"限定"。 栈是限定只能在表的一端进行插入和删除操作的线性表。 队列是限定只能在表的一端进行插入和在另一端进行删除操作的线性表。

\3. 遍历数据速度不同

standard 模式

这是默认模式，每次激活Activity时都会创建Activity实例，并放入任务栈中。使用场景：大多数Activity。

singleTop 模式

如果在任务的栈顶正好存在该Activity的实例，就重用该实例( 会调用实例的 onNewIntent() )，否则就会创建新的实例并放入栈顶，即使栈中已经存在该Activity的实例，只要不在栈顶，都会创建新的实例。使用场景如新闻类或者阅读类App的内容页面。

singleTask 模式

如果在栈中已经有该Activity的实例，就重用该实例(会调用实例的 onNewIntent() )。重用时，会让该实例回到栈顶，因此在它上面的实例将会被移出栈。如果栈中不存在该实例，将会创建新的实例放入栈中。使用场景如浏览器的主界面。不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面。

singleInstance 模式

在一个新栈中创建该Activity的实例，并让多个应用共享该栈中的该Activity实例。一旦该模式的Activity实例已经存在于某个栈中，任何应用再激活该Activity时都会重用该栈中的实例( 会调用实例的 onNewIntent() )。其效果相当于多个应用共享一个应用，不管谁激活该 Activity 都会进入同一个应用中。使用场景如闹铃提醒，将闹铃提醒与闹铃设置分离。singleInstance不要用于中间页面，如果用于中间页面，跳转会有问题，比如：A -> B (singleInstance) -> C，完全退出后，在此启动，首先打开的是B。



作者：王培921223
链接：https://www.jianshu.com/p/7661c292195a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### LaunchMode应用场景-百度-小米-乐视

standard，创建一个新的Activity。

singleTop，栈顶不是该类型的Activity，创建一个新的Activity。否则，onNewIntent。

singleTask，回退栈中没有该类型的Activity，创建Activity，否则，onNewIntent+ClearTop。

注意:

1. 设置了"singleTask"启动模式的Activity，它在启动的时候，会先在系统中查找属性值affinity等于它的属性值taskAffinity的Task存在； 如果存在这样的Task，它就会在这个Task中启动，否则就会在新的任务栈中启动。因此， 如果我们想要设置了"singleTask"启动模式的Activity在新的任务中启动，就要为它设置一个独立的taskAffinity属性值。
2. 如果设置了"singleTask"启动模式的Activity不是在新的任务中启动时，它会在已有的任务中查看是否已经存在相应的Activity实例， 如果存在，就会把位于这个Activity实例上面的Activity全部结束掉，即最终这个Activity 实例会位于任务的Stack顶端中。
3. 在一个任务栈中只有一个”singleTask”启动模式的Activity存在。他的上面可以有其他的Activity。这点与singleInstance是有区别的。

singleInstance，回退栈中，只有这一个Activity，没有其他Activity。

singleTop适合接收通知启动的内容显示页面。

例如，某个新闻客户端的新闻内容页面，如果收到10个新闻推送，每次都打开一个新闻内容页面是很烦人的。

singleTask适合作为程序入口点。

例如浏览器的主界面。不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面。

singleInstance应用场景：

闹铃的响铃界面。 你以前设置了一个闹铃：上午6点。在上午5点58分，你启动了闹铃设置界面，并按 Home 键回桌面；在上午5点59分时，你在微信和朋友聊天；在6点时，闹铃响了，并且弹出了一个对话框形式的 Activity(名为 AlarmAlertActivity) 提示你到6点了(这个 Activity 就是以 SingleInstance 加载模式打开的)，你按返回键，回到的是微信的聊天界面，这是因为 AlarmAlertActivity 所在的 Task 的栈只有他一个元素， 因此退出之后这个 Task 的栈空了。如果是以 SingleTask 打开 AlarmAlertActivity，那么当闹铃响了的时候，按返回键应该进入闹铃设置界面。



### activity启动模式（给例子具体分析，A(标准)-》B(单例)-》C(singleTop)-》D(singleTask)，分析有几个栈，每个栈内的activity）



### Activity 按 back 键退出，与强杀进程退出有啥区别？

**（1）应用被强杀**

- 整个App进程都被杀掉了，所有变量全都被清空，包括Application实例，更别提静态变量；
- 虽然变量被清空了，但 Activity 栈没有被清空，也就是说 A -> B -> C 这个栈还保存了，只是ABC 这几个 Activity 实例没有了。所以回到 App 时，显示的还是 C 页面。另外当 Activity 被强杀时，系统会调用 onSaveInstance 去让你保存一些变量；
- 当应用回到前台时，如果C页面中有静态变量或有些Application的全局变量，就NullPointer了；
- C页面不会正常走完生命周期onStop & onDestory

------

**（2）按 Back 键回退**

- 应用进程不会被杀掉；Activity 栈由 A -> B -> C 变成 A -> B；
- C页面会正常走完生命周期onStop & onDestory



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/9bfb74c50f6c
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。





### Activity A 跳转Activity B，Activity B再按back键回退，两个过程各自的生命周期

> - ActivityA跳转ActivityB的过程中,各自生命周期的执行顺序。例如：A.onCreate A.onStart A.onPause A.onStop B.onCreate B.onStart B.onPause B.onStop B.onDestroy?
>   ActivityA和ActivityB生命周期执行顺序如下： A.onPause －> B.onCreate －> B.onStart－> B.onResume－> A.onStop
> - ActivityB 按back键呢？
>   按下back键后： B.onPause－>A.onRestart－>A.onStart－>A.onResume－>B.onStop－>B.onDestory
> - ActivityB是个窗口Activity的情况下，1、2的结论呢？
>   若ActivityB是个窗口，ActivityA跳转到ActivityB时，ActivityA失去焦点部分可见，故不会调用onStop，此时生命周期顺序： A.onPause －> B.onCreate －> B.onStart－> B.onResume
>   按下Back键后：B.onPause－>A.onResume－>B.onStop－>B.onDestory
> - 切换横竖屏时，onCreate会调用吗？几次？
>   程序在运行时，一些设备的配置可能会改变，如：横竖屏的切换、键盘的可用性或语言的切换等，此时Activity会重新启动。其中的过程是：在销毁之前会先调用onSaveInstancestate()去保存应用中的一些数据，然后调用 onDestory()，最后才会去调用onCreate()或者onRestoreInstanceState方法重新启动Activiy。在切换屏幕时候会重新调用各个生命周期，切横屏时会执行一次onCreate，切竖屏时会执行两次onCreate。

1. 



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/5e5908ab3ea9
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### 子线程是否可以 context.startActivity() (如ApplicationContext), 会不会有什么问题？



### Application、Activity、Service 的创建顺序是什么？

[Application和四大组件启动时的方法顺序](https://www.jianshu.com/p/d52960e3f6f2)：

- 1.每个应用程序在第一次启动时，都会首先创建 Application 对象。创建时机是在ActivityThread.handleBindApplication方法中，创建 Application 时同时创建 ContextIml 实例；
- 2.之后通过 startActivity() 或 startActivityForResult() 请求启动一个 Activity 时，如果系统检测需要新建一个 Activity 对象，会调用到 ActivityThread.performLaunchActivity() 方法，创建一个Activity 实例时同时创建 ContextIml 实例；
- 3.通过 startService 或 bindService 时请求新创建一个 Service 对象，系统就会回调ActivityThread.handleCreateService() 完成相关数据操作，创建一个 Service 实例时同时创建ContextIml 实例
- 注：ContextIml 对象与 Application、Activity、Service 对象是一一绑定的



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/2c2abadae450
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### 如何跨进程拿 Context？如 Activity 还没启动的时候如何拿 Context？

**getApplication 与 getApplicationContext 区别**：getApplication() 用用来获取 Application实例的，但这个方法只有在 Activity 和 Service 中才能调用。如果在一些其它的场景，比如BroadcastReceiver 中也想获得 Application 的实例，这时需要借助 getApplicationContext() 方法。也就是说，getApplicationContext() 方法的作用域会更广一些，任何一个 Context 的实例，只要调用 getApplicationContext() 方法都可以拿到我们的Application对象；

**Application 中方法的执行顺序为**：Application 构造方法 → attachBaseContext() → onCreate()。如果在 attachBaseContext() 中或执行前，调用 getApplicationContext() 得到的值为null



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/2c2abadae450
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### 对Activity的理解是什么？

> 问题范围较大。从页面结构讲的 (Activity、Window、View、LayoutInflater)，用户交互入口

 



### Activity 的生命周期有哪些？

https://www.jianshu.com/p/6a9b727fc482

### Application启动流程



### AMS是如何确认Application启动完成的？关键条件是什么（zygote返给AMS的pid；应用的ActivityThread#main方法中会向AMS上报Application的binder对象）？



### Application#constructor、Application#onCreate、Application#attach他们的执行顺序（132）。Activity和Service呢？

- startActivity的具体过程。
- Activity#setContentView的具体过程。



### 谈谈你对Application的理解



### 为什么 Activity 间传递对象需要序列化？



### Activity 的启动流程是什么样的？



### 这一章主要讲解Activity相关的机制，包括Activity的启动流程，显示原理等相关面试问题，通过本章的学习，我们不但能熟悉它，更能深入了解它。

-  4-1 说说Activity的启动流程 (15:22)
-  4-2 说说Activity的显示原理 (14:59)
-  4-3 应用的UI线程是怎么启动的 (15:48)



### Android主线程

- Android主线程是怎么启动的



### Application, Activity, ContentProvider启动顺序

https://blog.csdn.net/beyond702/article/details/49666809



https://zhuanlan.zhihu.com/p/142910234





## Android中为什么主线程不会因为Looper.loop()里的死循环卡死？

https://www.zhihu.com/question/34652589/answer/90344494?from=profile_answer_card

http://gityuan.com/2016/03/18/start-activity-cycle/



## Fragment

### Fragment与Fragment、Activity通信的方式

1.直接在一个Fragment中调用另外一个Fragment中的方法

2.使用接口回调

3.使用广播

4.Fragment直接调用Activity中的public方法



### fragment周期，两个fragment切换周期变化，fragment通信



### Activity和fragment生命周期区别，fragment正常添加和viewpager添加的区别，fragment懒加载原理，FragmentPagerAdapter 和 FragmentStatePagerAdapter

















### Fragment生命周期

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master@master/img/20210319141506.png)



### Fragment 的理解，与 Activity 的区别





## Service

### Service生命周期？



### Activity与 Service 的区别？Service 一定没界面吗，Activity 一定有界面吗？

Activity 不是一定有界面。比如一个跳转逻辑控制类（机票的支付中间类）、透明页

[Service 也不是一定没界面](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5dbe43cf518825244b38a6c8)。Service 并不依赖于用户可视的 UI 界面，但这也不是绝对的，如前台 Service 就是与 Notification 界面结合使用的；Service 中也可以弹 Toast；

[Service中执行 LayoutInflate 是合法的](https://www.jianshu.com/p/94e0f9ab3f1d)，但是会使用系统默认的主题样式，如果你自定义了某些样式可能不会被使用。所以从理论上看也是可以有界面的



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/9bfb74c50f6c
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### service的 bindservice() 与 startService 区别

https://juejin.cn/post/6844903986131271688



### 本章主要讲除了Activity之外的应用组件相关面试问题，包括service的启动和绑定原理，静态广播和动态广播的注册和收发原理，provider的启动和数据传输原理等等。

-  5-1 说说service的启动原理 (13:56)
-  5-2 说说service的绑定原理-1 (12:46)
-  5-3 说说service的绑定原理-2 (11:03)
-  5-4 说说动态广播的注册和收发原理 (14:19)
-  5-5 说说静态广播的注册和收发原理 (21:40)
-  5-6 说说Provider的启动原理 (23:30)









## Intent的原理，作用，可以传递哪些类型的参数?





## ContentProvider

### 谈谈你对Android中Context的理解？四大组件里面的Context都来源于哪里。





### sqLite与contentProvider区别









## BroadCast



### BroadCast 的理解，分类有哪些？





## WindowMangerService中token到底是什么？有什么区别







## 四大组件底层的通信机制是怎样的？





## 在两个 Activity 之间传递对象还需要注意什么呢？

**对象的大小，对象的大小，对象的大小！！！**

重要的事情说三遍，一定要注意对象的大小。`Intent` 中的 `Bundle` 是使用 `Binder` 机制进行数据传送的。能使用的 Binder 的缓冲区是有大小限制的（有些手机是 2 M），而一个进程默认有 16 个 `Binder` 线程，所以一个线程能占用的缓冲区就更小了（ 有人以前做过测试，大约一个线程可以占用 128 KB）。所以当你看到 

`The Binder transaction failed because it was too large` 这类 `TransactionTooLargeException` 异常时，你应该知道怎么解决了。





## 谈谈你对Context的理解







## 疑问

- 为什么 Activity 间传递对象需要序列化？
- Activity 的启动流程是什么样的？
- 四大组件底层的通信机制是怎样的？
- AIDL 内部的实现原理是什么？
- 插件化编程技术应该从何学起？



