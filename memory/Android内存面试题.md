# LowMemoryKiller原理分析

http://gityuan.com/2016/09/17/android-lowmemorykiller/





# 面试题

## 一张图片100x100在内存中的大小？



## 内存泄露

### 什么情况导致内存泄漏-美团

1.资源对象没关闭造成的内存泄漏

描述： 资源性对象比如(Cursor，File文件等)往往都用了一些缓冲，我们在不使用的时候，应该及时关闭它们，以便它们的缓冲及时回收内存。它们的缓冲不仅存在于 java虚拟机内，还存在于java虚拟机外。如果我们仅仅是把它的引用设置为null,而不关闭它们，往往会造成内存泄漏。因为有些资源性对象，比如 SQLiteCursor(在析构函数finalize(),如果我们没有关闭它，它自己会调close()关闭)，如果我们没有关闭它，系统在回收它时也会关闭它，但是这样的效率太低了。因此对于资源性对象在不使用的时候，应该调用它的close()函数，将其关闭掉，然后才置为null.在我们的程序退出时一定要确保我们的资源性对象已经关闭。 程序中经常会进行查询数据库的操作，但是经常会有使用完毕Cursor后没有关闭的情况。如果我们的查询结果集比较小，对内存的消耗不容易被发现，只有在常时间大量操作的情况下才会复现内存问题，这样就会给以后的测试和问题排查带来困难和风险。

2.构造Adapter时，没有使用缓存的convertView

描述： 以构造ListView的BaseAdapter为例，在BaseAdapter中提供了方法： public View getView(int position, ViewconvertView, ViewGroup parent) 来向ListView提供每一个item所需要的view对象。初始时ListView会从BaseAdapter中根据当前的屏幕布局实例化一定数量的 view对象，同时ListView会将这些view对象缓存起来。当向上滚动ListView时，原先位于最上面的list item的view对象会被回收，然后被用来构造新出现的最下面的list item。这个构造过程就是由getView()方法完成的，getView()的第二个形参View convertView就是被缓存起来的list item的view对象(初始化时缓存中没有view对象则convertView是null)。由此可以看出，如果我们不去使用 convertView，而是每次都在getView()中重新实例化一个View对象的话，即浪费资源也浪费时间，也会使得内存占用越来越大。 ListView回收list item的view对象的过程可以查看: android.widget.AbsListView.java --> voidaddScrapView(View scrap) 方法。 示例代码：

```java
public View getView(int position, ViewconvertView, ViewGroup parent) {
View view = new Xxx(...); 
... ... 
return view; 
} 
```

修正示例代码：

```java
public View getView(int position, ViewconvertView, ViewGroup parent) {
View view = null; 
if (convertView != null) { 
view = convertView; 
populate(view, getItem(position)); 
... 
} else { 
view = new Xxx(...); 
... 
} 
return view; 
} 
```

3.Bitmap对象不在使用时调用recycle()释放内存

描述： 有时我们会手工的操作Bitmap对象，如果一个Bitmap对象比较占内存，当它不在被使用的时候，可以调用Bitmap.recycle()方法回收此对象的像素所占用的内存，但这不是必须的，视情况而定。可以看一下代码中的注释：

/** •Free up the memory associated with thisbitmap's pixels, and mark the •bitmap as "dead", meaning itwill throw an exception if getPixels() or •setPixels() is called, and will drawnothing. This operation cannot be •reversed, so it should only be called ifyou are sure there are no •further uses for the bitmap. This is anadvanced call, and normally need •not be called, since the normal GCprocess will free up this memory when •there are no more references to thisbitmap. */

4.试着使用关于application的context来替代和activity相关的context

这是一个很隐晦的内存泄漏的情况。有一种简单的方法来避免context相关的内存泄漏。最显著地一个是避免context逃出他自己的范围之外。使用Application context。这个context的生存周期和你的应用的生存周期一样长，而不是取决于activity的生存周期。如果你想保持一个长期生存的对象，并且这个对象需要一个context,记得使用application对象。你可以通过调用 Context.getApplicationContext() or Activity.getApplication()来获得。更多的请看这篇文章如何避免 Android内存泄漏。

5.注册没取消造成的内存泄漏

一些Android程序可能引用我们的Anroid程序的对象(比如注册机制)。即使我们的Android程序已经结束了，但是别的引用程序仍然还有对我们的Android程序的某个对象的引用，泄漏的内存依然不能被垃圾回收。调用registerReceiver后未调用unregisterReceiver。 比如:假设我们希望在锁屏界面(LockScreen)中，监听系统中的电话服务以获取一些信息(如信号强度等)，则可以在LockScreen中定义一个 PhoneStateListener的对象，同时将它注册到TelephonyManager服务中。对于LockScreen对象，当需要显示锁屏界面的时候就会创建一个LockScreen对象，而当锁屏界面消失的时候LockScreen对象就会被释放掉。 但是如果在释放 LockScreen对象的时候忘记取消我们之前注册的PhoneStateListener对象，则会导致LockScreen无法被垃圾回收。如果不断的使锁屏界面显示和消失，则最终会由于大量的LockScreen对象没有办法被回收而引起OutOfMemory,使得system_process 进程挂掉。 虽然有些系统程序，它本身好像是可以自动取消注册的(当然不及时)，但是我们还是应该在我们的程序中明确的取消注册，程序结束时应该把所有的注册都取消掉。

6.集合中对象没清理造成的内存泄漏

我们通常把一些对象的引用加入到了集合中，当我们不需要该对象时，并没有把它的引用从集合中清理掉，这样这个集合就会越来越大。如果这个集合是static的话，那情况就更严重了。



### 内存泄露的本质

无法回收无用的对象



### OOM，内存泄漏，内存溢出，java引用类型，ANR分析



### 构造一个内存泄露的场景







### 什么情况导致oom-乐视-美团

http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0920/3478.html

1. 使用更加轻量的数据结构
2. Android里面使用Enum
3. Bitmap对象的内存占用
4. 更大的图片
5. onDraw方法里面执行对象的创建
6. StringBuilder

### 造成 oom 的原因



### 下载一张很大的图，如何保证不 oom？ - [Android性能优化（五）之细说Bitmap](https://www.jianshu.com/p/e49ec7d053b3)





### 说一些引起内存泄漏的场景





### 检测到内存泄漏怎么修复





### 什么情况会导致内存泄漏，如何修复？



### 为什么会出现oom？

为了整个Android系统的内存控制需要，Android 系统为每一个应用程序都设置了一个硬性的Dalvik Heap Size 最大限制阈值，这个阈值在不同的设备上会因为 RAM 大小不同而各有差异。如果应用占用内存空间已经接近这个阈值，此时再尝试分配内存的话，很容易引起OutOfMemoryError 的错误。



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/2c2abadae450
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

### 哪些原因会导致 oom？

- **虚拟机堆内存不足**：内存泄漏（内存缓增）、大对象/大图片（内存突增）
- **内存碎片，无足够连续内存空间**：循环中创建对象、字符串拼接...
- **系统底层限制**：FD 数量超出限制、线程数量超出限制、其他系统限制



### debug 包有什么修改方式使不出现 oom？

Android为每个进程分配内存时，采用弹性的分配方式，即刚开始并不会给应用分配很多的内存，而是给每一个进程[分配一个“够用”的内存大小](https://www.jianshu.com/p/500ab0f48dc3)，这个值由具体的设备决定

在AndroidManifest.xml中的application标签中设置largeHeap为true，可以申请最多的内存的限制

这个内存限制的值是在 /system/build.prop文件中可以[查看与修改](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.droidviews.com%2Fedit-build-prop-file-on-android%2F)



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/2c2abadae450
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### 有哪些原因会引起内存泄漏？

https://www.jianshu.com/p/97fb764f2669



### 内存泄漏有什么方式检测？用过哪些工具，其中的原理是什么？

[Java内存问题 及 LeakCanary 原理分析](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5ab8d3d46fb9a028ca52f813)：

- 基本原理：用ActivityLifecycleCallbacks接口来检测Activity生命周期，主要是在**onDestroy()**方法中，手动调用 GC，然后利用ReferenceQueue+WeakReference 监听对象回收情况 ，来判断是否有释放不掉的引用，再结合dump memory的hpof文件, 用[HaHa](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fsquare%2Fhaha)分析出泄漏地方；
- LeakCanary会单独开一进程，用来执行分析任务，和监听任务分开处理。Application中可通过processName判断是否是任务执行进程；
- 利用主线程空闲的时候执行检测任务，在MessageQueue中加入了一个IdleHandler来得到主线程空闲回调；
- LeakCanary检测只针对Activiy里的相关对象。其他类无法使用，还得用MAT原始方法



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/2c2abadae450
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### Android内存泄露及管理

（1）内存溢出（OOM）和内存泄露（对象无法被回收）的区别。 

（2）引起内存泄露的原因

(3) 内存泄露检测工具 ------>LeakCanary



内存溢出 out of memory：是指程序在申请内存时，没有足够的内存空间供其使用，出现out of memory；比如申请了一个integer,但给它存了long才能存下的数，那就是内存溢出。内存溢出通俗的讲就是内存不够用。

内存泄露 memory leak：是指程序在申请内存后，无法释放已申请的内存空间，一次内存泄露危害可以忽略，但内存泄露堆积后果很严重，无论多少内存,迟早会被占光

内存泄露原因：

一、Handler 引起的内存泄漏。

解决：将Handler声明为静态内部类，就不会持有外部类SecondActivity的引用，其生命周期就和外部类无关，

如果Handler里面需要context的话，可以通过弱引用方式引用外部类

二、单例模式引起的内存泄漏。

解决：Context是ApplicationContext，由于ApplicationContext的生命周期是和app一致的，不会导致内存泄漏

三、非静态内部类创建静态实例引起的内存泄漏。

解决：把内部类修改为静态的就可以避免内存泄漏了

四、非静态匿名内部类引起的内存泄漏。

解决：将匿名内部类设置为静态的。

五、注册/反注册未成对使用引起的内存泄漏。

注册广播接受器、EventBus等，记得解绑。

六、资源对象没有关闭引起的内存泄漏。

在这些资源不使用的时候，记得调用相应的类似close（）、destroy（）、recycler（）、release（）等方法释放。

七、集合对象没有及时清理引起的内存泄漏。

通常会把一些对象装入到集合中，当不使用的时候一定要记得及时清理集合，让相关对象不再被引用。



作者：王培921223
链接：https://www.jianshu.com/p/7661c292195a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### 内存泄漏发生的情况有哪些？

答： 主要有四类情况

- 集合类泄漏
- 单例/静态变量造成的内存泄漏
- 匿名内部类/非静态内部类
- 资源未关闭造成的内存泄漏

具体解析如下：[内存泄漏三问—vivo真题](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/uZ61JhRowBlnNEzBic2_sQ)









## gc

### 垃圾回收

https://github.com/kesenhoo/android-training-course-in-chinese/blob/master/performance/memory.md

https://www.ibm.com/developerworks/cn/opensource/os-cn-android-mmry-rcycl/index.html

https://mp.weixin.qq.com/s/CUU3Ml394H_fkabhNNX32Q



### GC回收算法



### Java GC机制，为什么要执行 GC

> 那些不可能再被任何途径使用的对象，需要被回收，否则内存迟早都会被消耗空







### JVM,JMM,java加载对象的步骤，classLoader,GC回收算法



### java中GC 是如何判断对象是可以被回收的？

[Java 垃圾收集的原理：](https://links.jianshu.com/go?to=https%3A%2F%2Fapp.yinxiang.com%2FHome.action%23n%3D6add1e9f-f832-438c-ae3d-6f0f1fca8108%26s%3Ds38%26b%3Ded4692de-7e6e-4e29-ae3a-d4d01138be52%26ses%3D4%26sh%3D1%26sds%3D5%26)

- 自动垃圾收集的前提是清楚哪些内存可以被释放，主要有两个方面，最主要部分就是**对象实例**，存储在堆上的；另一个是**方法区中的元数据**等信息，例如类型不再使用，卸载该 Java 类比较合理；
- **对象实例收集**主要是两种基本算法，[引用计数](https://links.jianshu.com/go?to=https%3A%2F%2Fzh.wikipedia.org%2Fwiki%2F%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0)和可达性分析，**Java 选择的可达性分析**。**JVM 会把虚拟机栈和本地方法栈中**正在引用的对象**、**静态属性引用的对象**和**常量**，作为 GC Roots。



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/9bfb74c50f6c
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### JAVA GC原理

垃圾收集算法的核心思想是：对虚拟机可用内存空间，即堆空间中的对象进行识别，如果对象正在被引用，那么称其为存活对象

，反之，如果对象不再被引用，则为垃圾对象，可以回收其占据的空间，用于再分配。垃圾收集算法的选择和垃圾收集系统参数的合理调节直接影响着系统性能。



作者：王培921223
链接：https://www.jianshu.com/p/7661c292195a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



## JVM

### JVM的理解

http://www.infoq.com/cn/articles/java-memory-model-1



### jvm虚拟机，堆和栈的结构



### jvm虚拟机，堆和栈的结构，栈帧，JMM





### Java 中的几种引用类型，虚引用的使用场景？





### java的几种引用类型，弱引用的使用场景？





### JVM 堆内存溢出后，其他线程是否可继续工作？





## 面试题

https://coding.imooc.com/class/chapter/429.html#Anchor



## LeakCanary

### LeakCanary 实现原理

http://blog.csdn.net/cloud_huan/article/details/53081120



### LeakCanary的收集内存泄露是在 Activity 的什么时机，大致原理



## android虚拟机



### java虚拟机与Dalvik和ART区别

Java虚拟机：

1、java虚拟机基于栈。 基于栈的机器必须使用指令来载入和操作栈上数据，所需指令更多更多。

2、java虚拟机运行的是java字节码。（java类会被编译成一个或多个字节码.class文件）

Dalvik虚拟机：

1、dalvik虚拟机是基于寄存器的

2、Dalvik运行的是自定义的.dex字节码格式。（java类被编译成.class文件后，会通过一个dx工具将所有的.class文件转换成一个.dex文件，然后dalvik虚拟机会从其中读取指令和数据

3、常量池已被修改为只使用32位的索引，以 简化解释器。

4、一个应用，一个虚拟机实例，一个进程（所有android应用的线程都是对应一个linux线程，都运行在自己的沙盒中，不同的应用在不同的进程中运行。每个android dalvik应用程序都被赋予了一个独立的linux PID(app_*)）





作者：王培921223
链接：https://www.jianshu.com/p/7661c292195a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### Android虚拟机有哪些？区别是什么？

