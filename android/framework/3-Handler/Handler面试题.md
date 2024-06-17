# 面试题
![image-20210524184449133](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210630180039.png)

![image-20210722174217207](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210722174217.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303011508970.png)


https://www.bilibili.com/video/BV1eX4y1u7aY?p=1

https://www.bilibili.com/read/cv8323046



https://segmentfault.com/a/1190000022221446



https://www.jianshu.com/p/fec03e581e3e



## Handler连环27问，全都答上来就是P6！

https://www.jianshu.com/p/4a58568661e8



# Handler

- 9-2 说说threadLocal的原理 (11:57)

- 9-3 来说说looper的副业 (17:05)

- 9-4 怎么检查线程有耗时任务 (21:35)

- 9-5 怎么同步处理消息 (13:13)

  

## Handler 消息处理

 https://www.zhihu.com/question/34652589

https://github.com/xitu/gold-miner/blob/master/TODO/android-handler-internals.md

http://blog.csdn.net/guolin_blog/article/details/9991569

http://droidyue.com/blog/2015/11/08/make-use-of-handlerthread/

http://blog.csdn.net/luoshengyang/article/details/6817933



##  Handler怎么进行线程通信，原理是什么？





## Handler如果没有消息处理是阻塞的还是非阻塞的？

https://segmentfault.com/a/1190000022221446



## handler.post(Runnable) runnable是如何执行的？





## handler的Callback和handlemessage都存在，但callback返回true handleMessage还会执行么？





## Handler的sendMessage和postDelay的区别？



## Handler 队列阻塞算法、在Android中的地位、如何自己实现？
epoll机制，有数据读取，没有数据阻塞，让出 CPU 资源



http://blog.csdn.net/luoshengyang/article/details/6817933



## 讲一下handler，为啥没有卡主线程导致ANR？
epoll机制，有数据读取，没有数据阻塞，让出 CPU 资源




## handler原理及相关知识点，message回收策略

https://www.jianshu.com/p/fec03e581e3e



## handler原理及相关知识点，handler缓存池大小





## 如何修复匿名内部类 handler 造成的内存泄露？
[[3-内部类#内部类会造成程序的内存泄漏]]



## Handler的原理

Android中主线程是不能进行耗时操作的，子线程是不能进行更新UI的。所以就有了handler，它的作用就是实现线程之间的通信。

handler整个流程中，主要有四个对象，handler，Message,MessageQueue,Looper。当应用创建的时候，就会在主线程中创建handler对象，

我们通过要传送的消息保存到Message中，handler通过调用sendMessage方法将Message发送到MessageQueue中，Looper对象就会不断的调用loop()方法

不断的从MessageQueue中取出Message交给handler进行处理。从而实现线程之间的通信。




## handler介绍为什么阻塞不会造成anr屏障消息产生内存泄露原因handler内存泄露的引用链

https://www.zhihu.com/question/34652589/answer/90344494



**(1) Android中为什么主线程不会因为Looper.loop()里的死循环卡死？** 

这里涉及线程，先说说说进程/线程，**进程：** 每个app运行时前首先创建一个进程，该进程是由Zygote fork出来的，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，让App程序都是运行在Android Runtime。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。

**线程：** 线程对应用来说非常常见，比如每次new Thread().start都会创建一个新的线程。该线程与App所在进程之间资源共享，从Linux角度来说进程与线程除了是否共享资源外，并没有本质的区别，都是一个task_struct结构体，**在CPU看来进程或线程无非就是一段可执行的代码，CPU采用CFS调度算法，保证每个task都尽可能公平的享有CPU时间片**。

有了这么准备，再说说死循环问题：

对于线程既然是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？**简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，** 例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是**死循环又如何去处理其他事务呢？通过创建新线程的方式**。

真正会卡死主线程的操作是在回调方法onCreate/onStart/onResume等操作时间过长，会导致掉帧，甚至发生ANR，looper.loop本身不会导致应用卡死。

**主线程的死循环一直运行是不是特别消耗CPU资源呢？** 其实不然，这里就涉及到**Linux pipe/epoll机制**，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，详情见[Android消息机制1-Handler(Java层)](https://link.zhihu.com/?target=http%3A//www.yuanhh.com/2015/12/26/handler-message-framework/%23next)，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 **所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源**


## Handler实现机制，同步屏障，IdleHandler



## Handler如何实现对于消息的定时发送



## HandlerThread的实现原理



## intentService作用是什么,AIDL解决了什么问题-小米



## AsyncTask与HandlerThread区别？



## 应用层，消息的发送、接收、获取和处理；消息是如何存储的？延时消息一定准时么？是如何保证延时时间的？



## Handler#dispatchMessage细节，如何使用？



## handler里面消息有几种？普通消息、同步消息、消息屏障。如何使用？如何区分普通消息和异步消息？



## 如何实现给Handler发送一个Runnable，又不通过Handler#post(Runnable run)这个API？（Message#obj属性，或者通过反射设置Message#callback属性）



## Handler被设计出来的原因？有什么用？

一种东西被设计出来肯定就有它存在的意义，而`Handler`的意义就是切换线程。

作为`Android`消息机制的主要成员，它管理着所有与界面有关的消息事件，常见的使用场景有：

- 跨进程之后的界面消息处理。

比如Activity的启动，就是AMS在进行进程间通信的时候，通过Binder线程 将消息发送给`ApplicationThread`的消息处理者`Handler`，然后再将消息分发给主线程中去执行。

- 网络交互后切换到主线程进行UI更新

当子线程网络操作之后，需要切换到主线程进行UI更新。

总之一句话，`Hanlder`的存在就是为了解决在子线程中无法访问UI的问题。

## 为什么建议子线程不访问（更新）UI？

因为`Android`中的UI控件不是线程安全的，如果多线程访问UI控件那还不乱套了。

那为什么不加锁呢？

- `会降低UI访问的效率`。本身UI控件就是离用户比较近的一个组件，加锁之后自然会发生阻塞，那么UI访问的效率会降低，最终反应到用户端就是这个手机有点卡。
- `太复杂了`。本身UI访问时一个比较简单的操作逻辑，直接创建UI，修改UI即可。如果加锁之后就让这个UI访问的逻辑变得很复杂，没必要。

所以，Android设计出了 `单线程模型` 来处理UI操作，再搭配上Handler，是一个比较合适的解决方案。

## 子线程访问UI的 崩溃原因 和 解决办法？

崩溃发生在ViewRootImpl类的`checkThread`方法中：


```java
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }  

```

其实就是判断了当前线程 是否是 `ViewRootImpl`创建时候的线程，如果不是，就会崩溃。

而ViewRootImpl创建的时机就是界面被绘制的时候，也就是onResume之后，所以如果在子线程进行UI更新，就会发现当前线程（子线程）和View创建的线程（主线程）不是同一个线程，发生崩溃。

解决办法有三种：

- 在新建视图的线程进行这个视图的UI更新，主线程创建View，主线程更新View。
- 在`ViewRootImpl`创建之前进行子线程的UI更新，比如onCreate方法中进行子线程更新UI。
- 子线程切换到主线程进行UI更新，比如`Handler、view.post`方法。

## Handler 的 post(Runnable) 与 sendMessage 有什么区别

Hanlder中主要的发送消息可以分为两种：

- post(Runnable)
- sendMessage

```java
    public final boolean post(@NonNull Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

通过post的源码可知，其实`post和sendMessage`的区别就在于：

post方法给Message设置了一个`callback`。

那么这个callback有什么用呢？我们再转到消息处理的方法`dispatchMessage`中看看：


```csharp
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

    private static void handleCallback(Message message) {
        message.callback.run();
    }
复制代码
```

这段代码可以分为三部分看：

- 1、如果`msg.callback`不为空，也就是通过post方法发送消息的时候，会把消息交给这个msg.callback进行处理，然后就没有后续了。
- 2、如果`msg.callback`为空，也就是通过sendMessage发送消息的时候，会判断Handler当前的mCallback是否为空，如果不为空就交给Handler.Callback.handleMessage处理。
- 3、如果`mCallback.handleMessage`返回true，则无后续了。
- 4、如果`mCallback.handleMessage`返回false，则调用handler类重写的handleMessage方法。

所以post(Runnable) 与 sendMessage的区别就在于后续消息的处理方式，是交给`msg.callback`还是 `Handler.Callback`或者`Handler.handleMessage`。

## Handler.Callback.handleMessage 和 Handler.handleMessage 有什么不一样？为什么这么设计？

接着上面的代码说，这两个处理方法的区别在于`Handler.Callback.handleMessage`方法是否返回true：

- 如果为`true`，则不再执行Handler.handleMessage
- 如果为`false`，则两个方法都要执行。

那么什么时候有`Callback`，什么时候没有呢？这涉及到两种Hanlder的 创建方式：



```kotlin
    val handler1= object : Handler(){
        override fun handleMessage(msg: Message) {
            super.handleMessage(msg)
        }
    }

    val handler2 = Handler(object : Handler.Callback {
        override fun handleMessage(msg: Message): Boolean {
            return true
        }
    })

```

常用的方法就是第1种，派生一个Handler的子类并重写handleMessage方法。 而第2种就是系统给我们提供了一种不需要派生子类的使用方法，只需要传入一个Callback即可。



## 15、Handler里面藏着的CallBack能做什么？

答: 利用此CallBack拦截Handler的消息处理

在上一篇中我们分析到，`dispatchMessage`方法的处理步骤:

1、首先，检查Message的callback是否为null，不为null就通过`handleCallBack`来处理消息，Message的callback是一个Runnable对象，实际上就是Handler的`post`系列方法所传递的Runnable参数

2、其次，检查Handler里面藏着的CallBack是否为null，不为null就调用mCallback的`handleMessage`方法来处理消息，并判断其返回值：为true，那么 Handler 的 `handleMessage(msg)` 方法就不会被调用了；为false，那么就意味着**一个消息可以同时被 Callback 以及 Handler 处理**。

3、最后，调用Handler的`handleMessage`方法来处理消息

通过上面分析我们知道Handler处理消息的顺序是：**Message的Callback > Handler的Callback > Handler的`handleMessage`方法**

使用场景: Hook ActivityThread.mH ， 在 ActivityThread 中有个成员变量 `mH` ，它是个 Handler，又是个极其重要的类，几乎所有的插件化框架都使用了这个方法



## ActivityThread中做了哪些关于Handler的工作？（为什么主线程不需要单独创建Looper）

主要做了两件事：
- 1、在main方法中，创建了主线程的`Looper`和`MessageQueue`，并且调用loop方法开启了主线程的消息循环。

```java
public static void main(String[] args) {

        Looper.prepareMainLooper();

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

```

- 2、创建了一个Handler来进行四大组件的启动停止等事件处理

```java
final H mH = new H();

class H extends Handler {
        public static final int BIND_APPLICATION        = 110;
        public static final int EXIT_APPLICATION        = 111;
        public static final int RECEIVER                = 113;
        public static final int CREATE_SERVICE          = 114;
        public static final int STOP_SERVICE            = 116;
        public static final int BIND_SERVICE            = 121;

```

## 子线程使用Handler及相关注意事项

我们通常使用Handler都是从子线程发送消息到主线程去处理，那么这里我们尝试一下从主线程发送消息到子线程来处理，上代码：



```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //创建线程实例并开启
        MyThread myThread = new MyThread();
        myThread.start();
        //打开这段注释就不会crash，且看下面分析
//      try {
//          Thread.sleep(500);
//      } catch (InterruptedException e) {
//          e.printStackTrace();
//      }
        //获取Handler发送消息
        myThread.getHandler().sendEmptyMessage(0x001);
    }

    public static class MyThread extends Thread {
        private Handler mHandler;

        public void run() {
            Looper.prepare();
            mHandler = new Handler() {
                public void handleMessage(@NonNull Message msg) {
                    if(msg.what == 0x001){
                        Log.d("print", "handleMessage: ");
                    }
                }
            };
            Looper.loop();
        }

        public Handler getHandler(){
            return mHandler;
        }
    }
}
```

运行一下上述代码，发现会Crash，如下图：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210629184434)

报了一个空指针异常，原因就是多线程并发，当主线程执行到sendEnptyMessage时，子线程的Handler还没有创建。因此我们可以在获取Handler的时候让主线程休眠一下在执行，应用就不会Crash了，打开上面代码的注释即可

**值得注意的是**：我们自己创建的Looper在使用完毕后应该调用`quit`方法来终止消息循环，如果不退出的话，那么该线程的Looper处理完所有的消息后，就会处于一个阻塞状态，要知道线程是比较重量级的，如果一直存在，肯定会对应用性能造成一定的影响。而如果退出Looper，这个线程就会立刻终止，因此建议不需要的时候终止Looper。

因此在子线程使用Handler，我们需要注意一下两点：

**1、必须调用`Looper.prepare()`创建当前线程的 Looper，并调用`Looper.loop()`开启消息循环**

**2、必须在使用结束后调用Looper的`quit`方法退出当前线程**



## 1、Handler有哪些作用?

答：

1、Handler能够进行线程之间的切换

2、Handler能够按照顺序处理消息，避免并发

3、Handler能够阻塞线程

4、Handler能够发送并处理延迟消息

解析:

1、Handler能够进行线程之间的切换，是因为使用了不同线程的Looper处理消息

2、Handler能够按照顺序处理消息，避免并发，是因为消息在入队的时候会按照时间升序对当前链表进行排序，Looper读取的时候，MessageQueue的`next`方法会循环加锁，同时配合阻塞唤醒机制

3、Handler能够阻塞线程主要是基于Linux的epoll机制实现的

4、Handler能够处理延迟消息，是因为MessageQueue的`next`方法中会拿当前消息时间和当前时间做比较，如果是延迟消息，那么就会阻塞当前线程，等阻塞时间到，在执行该消息

## 2、为什么我们能在主线程直接使用Handler，而不需要创建Looper？

答：主线程已经创建了Looper，并开启了消息循环

## 3、如果想要在子线程创建Handler，需要做什么准备？

答：需要先创建Looper，并开启消息循环

## 4、一个线程有几个Handler？

答：可以有任意多个

## 5、一个线程有几个Looper？如何保证？

答：一个线程只有一个Looper，通过ThreadLocal来保证

## 6、Handler发送消息的时候，时间为啥要取SystemClock.uptimeMillis() + delayMillis，可以把SystemClock.uptimeMillis() 换成System.currentTimeMillis()吗？

答：不可以

**SystemClock.uptimeMillis()** 这个方法获取的时间，是自系统开机到现在的一个毫秒数，这个时间是个相对的

**System.currentTimeMillis()** 这个方法获取的是自**1970-01-01 00:00:00** 到现在的一个毫秒数，这是一个和系统强关联的时间，而且这个值可以做修改

1、使用System.currentTimeMillis()可能会导致延迟消息失效

2、最终这个时间会被设置到Message的when属性，而Message的when属性只是需要一个时间差来表示消息的先后顺序，使用一个相对时间就行了，没必要使用一个绝对时间



## 8、Handler内存泄露原因? 如何解决？

内存泄漏的本质是长生命周期的对象持有短生命周期对象的引用，导致短生命周期的对象无法被回收，从而导致了内存泄漏

下面我们就看个导致内存泄漏的例子



```java
public class MainActivity extends AppCompatActivity {

    private final Handler mHandler = new Handler() {
        @Override
        public void handleMessage(@NonNull Message msg) {
           //do something
        }
    };
  
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //发送一个延迟消息，10分钟后在执行
        mHandler.sendEmptyMessageDelayed(0x001,10*60*1000);
    }
}
```

上述代码:

1、我们通过匿名内部类的方式创建了一个Handler的实例

2、在`onCreate`方法里面通过Handler实例发送了一个延迟10分钟执行的消息

我们发送的这个延迟10分钟执行的消息它是持有Handler的引用的，根据Java特性我们又知道，非静态内部类会持有外部类的引用，因此当前Handler又持有Activity的引用，而Message又存在MessageQueue中，MessageQueue又在当前线程中，因此会存在一个引用链关系:

**当前线程->MessageQueue->Message->Handler->Activity**

因此当我们退出Activity的时候，由于消息需要在10分钟后在执行，因此会一直持有Activity，从而导致了Activity的内存泄漏

通过上面分析我们知道了内存泄漏的原因就是持有了Activity的引用，那我们是不是会想，切断这条引用，那么如果我们需要用到Activity相关的属性和方法采用弱引用的方式不就可以了么？我们实际操作一下，把Handler写成一个静态内部类



```java
public class MainActivity extends AppCompatActivity {

    private final SafeHandler mSafeHandler = new SafeHandler(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //发送一个延迟消息，10分钟后在执行
        mSafeHandler.sendEmptyMessageDelayed(0x001,10*60*1000);
    }

    //静态内部类并持有Activity的弱引用
    private static class SafeHandler extends Handler{
      
        private final WeakReference<MainActivity> mWeakReference;
      
        public SafeHandler(MainActivity activity){
            mWeakReference = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(@NonNull Message msg) {
            MainActivity mMainActivity = mWeakReference.get();
            if(mMainActivity != null){
                //do something
            }
        }
    }
}
```

上述代码

1、把Handler定义成了一个静态内部类，并持有当前Activity的弱引用，弱引用会在Java虚拟机发生gc的时候把对象给回收掉

经过上述改造，我们解决了Activity的内存泄漏，此时的引用链关系为:

**当前线程->MessageQueue->Message->Handler**

我们会发现Message还是会持有Handler的引用，从而导致Handler也会内存泄漏，所以我们应该在Activity销毁的时候，在他的生命周期方法里，把MessageQueue中的Message都给移除掉，因此最终就变成了这样：



```java
public class MainActivity extends AppCompatActivity {

    private final SafeHandler mSafeHandler = new SafeHandler(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //发送一个延迟消息，10分钟后在执行
        mSafeHandler.sendEmptyMessageDelayed(0x001,10*60*1000);
    }
  
    @Override
    protected void onDestroy() {
        super.onDestroy();
        mSafeHandler.removeCallbacksAndMessages(null);
    }

    //静态内部类并持有Activity的弱引用
    private static class SafeHandler extends Handler{
      
        private final WeakReference<MainActivity> mWeakReference;
      
        public SafeHandler(MainActivity activity){
            mWeakReference = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(@NonNull Message msg) {
            MainActivity mMainActivity = mWeakReference.get();
            if(mMainActivity != null){
                //do something
            }
        }
    }
}
```

因此当Activity销毁后，引用链关系为:

**当前线程->MessageQueue**

而当前线程和MessageQueue的生命周期和应用生命周期是一样长的，因此也就不存在内存泄漏了，完美。

所以解决Handler内存泄漏最好的方式就是：**将Handler定义成静态内部类，内部持有Activity的弱引用，并在Activity销毁的时候移除所有消息**



## 13、Handler是如何进行线程切换的呢？

答：使用不同线程的Looper处理消息

我们通常处理消息是在Handler的`handleMessage`方法中，那么这个方法是在哪里回调的呢？看下面这段代码



```java
public static void loop() {
    //开启死循环读取消息
    for (;;) {
         // 调用Message对应的Handler处理消息
         msg.target.dispatchMessage(msg);
    }
}
```

上述代码中`msg.target`其实就是我们发送消息的Handler，因此他会回调Handler的`dispatchMessage`方法，而`dispatchMessage`这个方法我们在上一篇中重点分析过，其中有一部分逻辑就是会回调到Handler的`handleMessage`方法，我们还可以发现，Handler的`handleMessage`方法所在的线程是由Looper的`loop`方法决定的。平时我们使用的时候，是从异步线程发送消息到 Handler，而这个 Handler 的 `handleMessage()` 方法是在主线程调用的，因为Looper是在主线程创建的，所以消息就从异步线程切换到了主线程。



## Handler内存泄露，最终是谁持有的activity？

https://juejin.cn/post/6844903834301497352#heading-5

a、内部类持有外部类引用

![image-20210708153908886](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708153908.png)





## Handler如何处理发送延迟消息？

由底层epoll机制延迟实现

nativePollOnce    //block阻塞

nativeWake

![image-20210708153954547](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708153954.png)



## Handler价值：卡顿监控





## Handler能做定时器嘛？

不可以，因为每个消息执行时间不定，等到自己发送的消息执行的时间就不确定。（handler机制导致不准确，不是nativePollOnce导致不准确）



# Looper



## 初始化

### ThreadLocal的原理，以及在Looper是如何应用的？



### ThreadLocal运行机制？这种机制设计的好处？

下面就具体说说`ThreadLocal`运行机制。



```kotlin
//ThreadLocal.java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
复制代码
```

从`ThreadLocal`类中的get和set方法可以大致看出来，有一个`ThreadLocalMap`变量，这个变量存储着键值对形式的数据。

- `key`为this，也就是当前ThreadLocal变量。
- `value`为T，也就是要存储的值。

然后继续看看`ThreadLocalMap`哪来的，也就是getMap方法：



```csharp
    //ThreadLocal.java
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    //Thread.java
    ThreadLocal.ThreadLocalMap threadLocals = null;
复制代码
```

原来这个`ThreadLocalMap`变量是存储在线程类Thread中的。

所以`ThreadLocal`的基本机制就搞清楚了：

在每个线程中都有一个threadLocals变量，这个变量存储着ThreadLocal和对应的需要保存的对象。

这样带来的好处就是，在不同的线程，访问同一个ThreadLocal对象，但是能获取到的值却不一样。

挺神奇的是不是，其实就是其内部获取到的Map不同，Map和Thread绑定，所以虽然访问的是同一个`ThreadLocal`对象，但是访问的Map却不是同一个，所以取得值也不一样。

这样做有什么好处呢？为什么不直接用Map存储线程和对象呢？

**打个比方：**

![img](https:////upload-images.jianshu.io/upload_images/25211949-ba8807eaadb24989.image?imageMogr2/auto-orient/strip|imageView2/2/w/400/format/webp)

- `ThreadLocal`就是老师。
- `Thread`就是同学。
- `Looper`（需要的值）就是铅笔。

现在老师买了一批铅笔，然后想把这些铅笔发给同学们，怎么发呢？两种办法：

- 1、老师把每个铅笔上写好每个同学的名字，放到一个大盒子里面去（map），用的时候就让同学们自己来找。

这种做法就是Map里面存储的是`同学和铅笔`，然后用的时候通过同学来从这个Map里找铅笔。

这种做法就有点像使用一个Map，存储所有的线程和对象，不好的地方就在于会很混乱，每个线程之间有了联系，也容易造成内存泄漏。

- 2、老师把每个铅笔直接发给每个同学，放到同学的口袋里（map），用的时候每个同学从口袋里面拿出铅笔就可以了。

这种做法就是Map里面存储的是`老师和铅笔`，然后用的时候老师说一声，同学只需要从口袋里拿出来就行了。

很明显这种做法更科学，这也就是`ThreadLocal`的做法，因为铅笔本身就是同学自己在用，所以一开始就把铅笔交给同学自己保管是最好的，每个同学之间进行隔离。

### 还有哪些地方运用到了ThreadLocal机制？

比如：Choreographer。

```java
public final class Choreographer {

    // Thread local storage for the choreographer.
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
            if (looper == Looper.getMainLooper()) {
                mMainInstance = choreographer;
            }
            return choreographer;
        }
    };

    private static volatile Choreographer mMainInstance;


```

`Choreographer`主要是主线程用的，用于配合 `VSYNC` 中断信号。

所以这里使用`ThreadLocal`更多的意义在于完成线程单例的功能。


### Looper、handler、线程间的关系。例如一个线程可以有几个Looper可以对应几个Handler？

https://blog.csdn.net/ly502541243/article/details/87475229

https://www.jianshu.com/p/859f455f03d7



> 由于使用了ThreadLocal机制，所以注定了一个线程只能有一个Looper，但Handler可以new无数个。



### Handler 机制中是怎么保证每个线程的 Looper 是唯一的？

https://blog.csdn.net/ly502541243/article/details/87475229



### Looper可以在子线程创建吗





### 建立起消息循环,需要线程中调用Looper.loop()

[Thread]

```java
public class HandlerThread extends Thread{
 @Override
    public void run() {
        Looper.prepare();
        Looper.loop();
    }
}
```



### 可以多次创建Looper吗？

Looper的创建是通过`Looper.prepare`方法实现的，而在prepare方法中就判断了，当前线程是否存在Looper对象，如果有，就会直接抛出异常：

```csharp
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
复制代码
```

所以同一个线程，只能创建一个`Looper`，多次创建会报错。


## 获取

### Looper是干嘛呢？怎么获取当前线程的Looper？为什么不直接用Map存储线程和对象呢？

在Handler发送消息之后，消息就被存储到`MessageQueue`中，而`Looper`就是一个管理消息队列的角色。 Looper会从`MessageQueue`中不断的查找消息，也就是loop方法，并将消息交回给Handler进行处理。

而Looper的获取就是通过`ThreadLocal`机制:



```csharp
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
复制代码
```

通过`prepare`方法创建Looper并且加入到sThreadLocal中，通过`myLooper`方法从sThreadLocal中获取Looper。





## 退出

### Looper中的quitAllowed字段是啥？有什么用？

按照字面意思就是是否允许退出，我们看看他都在哪些地方用到了：



```java
    void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }
        }
    }
复制代码
```

哦，就是这个`quit`方法用到了，如果这个字段为`false`，代表不允许退出，就会报错。

但是这个`quit`方法又是干嘛的呢？从来没用过呢。 还有这个`safe`又是啥呢？

其实看名字就差不多能了解了，quit方法就是退出消息队列，终止消息循环。

- 首先设置了`mQuitting`字段为true。
- 然后判断是否安全退出，如果安全退出，就执行`removeAllFutureMessagesLocked`方法，它内部的逻辑是清空所有的延迟消息，之前没处理的非延迟消息还是需要取处理，然后设置非延迟消息的下一个节点为空（p.next=null）。
- 如果不是安全退出，就执行`removeAllMessagesLocked`方法，直接清空所有的消息，然后设置消息队列指向空（mMessages = null）

然后看看当调用quit方法之后，消息的发送和处理：



```java
//消息发送
    boolean enqueueMessage(Message msg, long when) {
        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
        }
复制代码
```

当调用了quit方法之后，`mQuitting`为true，消息就发不出去了，会报错。

再看看消息的处理，loop和next方法：



```java
    Message next() {
        for (;;) {
            synchronized (this) {
                if (mQuitting) {
                    dispose();
                    return null;
                } 
            }  
        }
    }

    public static void loop() {
        for (;;) {
            Message msg = queue.next();
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
        }
    }
复制代码
```

很明显，当`mQuitting`为true的时候，next方法返回null，那么loop方法中就会退出死循环。

那么这个`quit`方法一般是什么时候使用呢？

- 主线程中，一般情况下肯定不能退出，因为退出后主线程就停止了。所以是当APP需要退出的时候，就会调用quit方法，涉及到的消息是EXIT_APPLICATION，大家可以搜索下。
- 子线程中，如果消息都处理完了，就需要调用quit方法停止消息循环。

## loop循环

### Looper.loop方法是死循环，为什么不会卡死（ANR）？

我大致总结下：

- 1、主线程本身就是需要一只运行的，因为要处理各个View，界面变化。所以需要这个死循环来保证主线程一直执行下去，不会被退出。
- 2、真正会卡死的操作是在某个消息处理的时候操作时间过长，导致掉帧、ANR，而不是loop方法本身。
- 3、在主线程以外，会有其他的线程来处理接受其他进程的事件，比如`Binder线程（ApplicationThread）`，会接受AMS发送来的事件
- 4、在收到跨进程消息后，会交给主线程的`Hanlder`再进行消息分发。所以Activity的生命周期都是依靠主线程的`Looper.loop`，当收到不同Message时则采用相应措施，比如收到`msg=H.LAUNCH_ACTIVITY`，则调用`ActivityThread.handleLaunchActivity()`方法，最终执行到onCreate方法。
- 5、当没有消息的时候，会阻塞在loop的`queue.next()`中的`nativePollOnce()`方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生。所以死循环也不会特别消耗CPU资源。



我们看到 ActivityThread 中，它实际上是有一个 handleMessage 方法，其实 ActivityThread 就是一个 Handler，我们在使用的过程中的很多事件（如 Activity、Service 的各种生命周期）都在这里的各种 Case 中，也就是说我们平时说的主线程其实就是依靠这个 Looper 的 loop 方法来处理各种消息，从而实现如 Activity 的声明周期的回调等等的处理，从而回调给我们使用者。

```
public void handleMessage(Message msg) {

            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));

            switch (msg.what) {

                case XXX:

                // ...

            }

            Object obj = msg.obj;

            if (obj instanceof SomeArgs) {

                ((SomeArgs) obj).recycle();

            }

            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));

        }

    }
```

因此不能说主线程不会阻塞，因为主线程本身就是阻塞的，其中所有事件都由主线程进行处理，从而使得我们能在这个循环的过程中作出自己的各种处理（如 View 的绘制等）。

而这个问题的意思应该是为何这样一个死循环不会使得界面卡顿，这有两个原因：

1. 界面的绘制本身就是这个循环内的一个事件
2. 界面的绘制是通过了同步屏障保护下发送的异步消息，会被主线程优先处理，因此使得界面绘制拥有了最高的优先级，不会因为 Handler 中事件太多而造成卡顿。


### 9、线程维护的Looper，在消息队列无消息时的处理方案是什么？有什么用？

答：当消息队列无消息时，Looper会阻塞当前线程，释放cpu资源，提高App性能

我们知道Looper的`loop`方法中有个死循环一直在读取MessageQueue中的消息，其实是调用了MessageQueue中的`next`方法，这个方法会在无消息时，调用Linux的epoll机制，使得线程进入阻塞状态，当有新消息到来时，就会将它唤醒，next方法里会判断当前消息是否是延迟消息，如果是则阻塞线程，如果不是，则会返回这条消息并将其从优先级队列中给移除



### 16、Handler阻塞唤醒机制是怎么一回事？

答： Handler的阻塞唤醒机制是基于Linux的阻塞唤醒机制。

这个机制也是类似于handler机制的模式。在本地创建一个文件描述符，然后需要等待的一方则监听这个文件描述符，唤醒的一方只需要修改这个文件，那么等待的一方就会收到文件从而打破唤醒。和Looper监听MessageQueue，Handler添加message是比较类似的。具体的Linux层知识读者可通过这篇文章详细了解（[传送门](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FYlc5mPwMzWoK2CIthZy0Vw)）

https://mp.weixin.qq.com/s/Ylc5mPwMzWoK2CIthZy0Vw





### 7、为什么Looper死循环，却不会导致应用卡死？

答：因为当Looper处理完所有消息的时候，会调用Linux的epoll机制进入到阻塞状态，当有新的Message进来的时候会打破阻塞继续执行。

应用卡死即ANR: 全称Applicationn Not Responding，中文意思是应用无响应，当我发送一个消息到主线程，Handler经过一定时间没有执行完这条消息，那么这个时候就会抛出ANR异常

Looper死循环: 循环执行各种事务，Looper死循环说明线程还活着，如果没有Looper死循环，线程结束，应用就退出了，当Looper处理完所有消息的时候会调用Linux的epoll机制进入到阻塞状态，当有新的Message进来的时候会打破阻塞继续执行



### 妙用 Looper 机制

1、我们可以通过Looper`getMainLooper`方法获取主线程Looper，从而可以判断当前线程是否是主线程

2、将 Runnable post 到主线程执行



```java
public final class MainThread {

    private MainThread() {
    }

    private static final Handler HANDLER = new Handler(Looper.getMainLooper());

    public static void run(@NonNull Runnable runnable) {
        if (isMainThread()) {
            runnable.run();
        }else{
            HANDLER.post(runnable);
        }
    }

    public static boolean isMainThread() {
        return Looper.myLooper() == Looper.getMainLooper();
    }
}
```















## Questions

 https://segmentfault.com/a/1190000022221446



1. Looper 死循环为什么不会导致应用卡死，会消耗大量资源吗？
2. 主线程的消息循环机制是什么（死循环如何处理其它事务）？
3. ActivityThread 的动力是什么？（ActivityThread执行Looper的线程是什么）
4. Handler 是如何能够线程切换，发送Message的？（线程间通讯）
5. 子线程有哪些更新UI的方法。
6. 子线程中Toast，showDialog，的方法。（和子线程不能更新UI有关吗）
7. 如何处理Handler 使用不当导致的内存泄露？



## Handler 8问

https://www.jianshu.com/p/a0a54ee67e4f

https://www.jianshu.com/p/4a58568661e8

https://www.jianshu.com/p/57b7c6da49f1

https://www.jianshu.com/p/08e26678044d



# MessageQueue

### MessageQueue是干嘛呢？用的什么数据结构来存储数据？

看名字应该是个队列结构，队列的特点是什么？`先进先出`，一般在队尾增加数据，在队首进行取数据或者删除数据。

那`Hanlder`中的消息似乎也满足这样的特点，先发的消息肯定就会先被处理。但是，`Handler`中还有比较特殊的情况，比如延时消息。

延时消息的存在就让这个队列有些特殊性了，并不能完全保证先进先出，而是需要根据时间来判断，所以`Android`中采用了链表的形式来实现这个队列，也方便了数据的插入。

来一起看看消息的发送过程，无论是哪种方法发送消息，都会走到`sendMessageDelayed`方法



```java
    public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        return enqueueMessage(queue, msg, uptimeMillis);
    }
复制代码
```

`sendMessageDelayed`方法主要计算了消息需要被处理的时间，如果`delayMillis`为0，那么消息的处理时间就是当前时间。

然后就是关键方法`enqueueMessage`。



```kotlin
    boolean enqueueMessage(Message msg, long when) {
        synchronized (this) {
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; 
                prev.next = msg;
            }

            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
复制代码
```

不懂得地方先不看，只看我们想看的：

- 首先设置了`Message`的when字段，也就是代表了这个消息的处理时间
- 然后判断当前队列是不是为空，是不是即时消息，是不是执行时间when大于表头的消息时间，满足任意一个，就把当前消息msg插入到表头。
- 否则，就需要遍历这个队列，也就是`链表`，找出when小于某个节点的when，找到后插入。

好了，其他内容暂且不看，总之，插入消息就是通过消息的执行时间，也就是`when`字段，来找到合适的位置插入链表。

具体方法就是通过死循环，使用快慢指针p和prev，每次向后移动一格，直到找到某个节点p的when大于我们要插入消息的when字段，则插入到p和prev之间。 或者遍历到链表结束，插入到链表结尾。

所以，`MessageQueue`就是一个用于存储消息、用链表实现的特殊队列结构。

### 延迟消息是怎么实现的？

总结上述内容，延迟消息的实现主要跟消息的统一存储方法有关，也就是上文说过的`enqueueMessage`方法。

无论是即时消息还是延迟消息，都是计算出具体的时间，然后作为消息的when字段进程赋值。

然后在MessageQueue中找到合适的位置（安排when小到大排列），并将消息插入到`MessageQueue`中。

这样，`MessageQueue`就是一个按照消息时间排列的一个链表结构。

### MessageQueue的消息怎么被取出来的？

刚才说过了消息的存储，接下来看看消息的取出，也就是`queue.next`方法。



```csharp
    Message next() {
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
            }
        }
    }
复制代码
```

奇怪，为什么取消息也是用的死循环呢？

其实死循环就是为了保证一定要返回一条消息，如果没有可用消息，那么就阻塞在这里，一直到有新消息的到来。

其中，`nativePollOnce`方法就是阻塞方法，`nextPollTimeoutMillis`参数就是阻塞的时间。

那什么时候会阻塞呢？两种情况：

- 1、有消息，但是当前时间小于消息执行时间，也就是代码中的这一句：



```csharp
if (now < msg.when) {
    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
}
复制代码
```

这时候阻塞时间就是消息时间减去当前时间，然后进入下一次循环，阻塞。

- 2、没有消息的时候，也就是上述代码的最后一句：



```csharp
if (msg != null) {} 
    else {
    // No more messages.
    nextPollTimeoutMillis = -1;
    }
复制代码
```

`-1`就代表一直阻塞。

### MessageQueue没有消息时候会怎样？阻塞之后怎么唤醒呢？说说pipe/epoll机制？

接着上文的逻辑，当消息不可用或者没有消息的时候就会阻塞在next方法，而阻塞的办法是通过pipe/epoll机制

`epoll机制`是一种IO多路复用的机制，具体逻辑就是一个进程可以监视多个描述符，当某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作，这个读写操作是阻塞的。在Android中，会创建一个`Linux管道（Pipe）`来处理阻塞和唤醒。

- 当消息队列为空，管道的读端等待管道中有新内容可读，就会通过`epoll`机制进入阻塞状态。
- 当有消息要处理，就会通过管道的写端写入内容，唤醒主线程。

**那什么时候会怎么唤醒消息队列线程呢？**

还记得刚才插入消息的`enqueueMessage`方法中有个`needWake`字段吗，很明显，这个就是表示是否唤醒的字段。

其中还有个字段是`mBlocked`，看字面意思是阻塞的意思，去代码里面找找：



```kotlin
Message next() {
        for (;;) {
            synchronized (this) {
                if (msg != null) {
                    if (now < msg.when) {
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        return msg;
                    }
                } 
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
            }
        }
    }
复制代码
```

在获取消息的方法`next`中，有两个地方对`mBlocked`赋值：

- 当获取到消息的时候，`mBlocked`赋值为`false`，表示不阻塞。
- 当没有消息要处理，也没有`idleHandler`要处理的时候，`mBlocked`赋值为`true`，表示阻塞。

好了，确实这个字段就表示是否阻塞的意思，再去看看`enqueueMessage`方法中，唤醒机制：



```kotlin
    boolean enqueueMessage(Message msg, long when) {
        synchronized (this) {
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; 
                prev.next = msg;
            }

            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
复制代码
```

- 当链表为空或者时间小于表头消息时间，那么就插入表头，并且设置是否唤醒为`mBlocked`。

再结合上述的例子，也就是当有新消息要插入表头了，这时候如果之前是阻塞状态（mBlocked=true），那么就要唤醒线程了。

- 否则，就需要取链表中找到某个节点并插入消息，在这之前需要赋值

  ```
  needWake = mBlocked && p.target == null && msg.isAsynchronous()
  ```

也就是在插入消息之前，需要判断是否阻塞，并且表头是不是屏障消息，并且当前消息是不是异步消息。 也就是如果现在是同步屏障模式下，那么要插入的消息又刚好是异步消息，那就不用管插入消息问题了，直接唤醒线程，因为异步消息需要先执行。

- 最后一点，是在循环里，如果发现之前就存在异步消息，那就还是设置是否唤醒为`false`。

意思就是，如果之前有异步消息了，那肯定之前就唤醒过了，这时候就不需要再次唤醒了。

最后根据`needWake`的值，决定是否调用`nativeWake`方法唤醒`next()`方法。



### 10、MessageQueue什么情况下会被唤醒？

答：需要分情况

1、发送消息过来，此时MessageQueue中无消息或者当前发送过来的消息携带的when为0或者有延迟执行的消息，那么需要唤醒

2、当遇到同步屏障且当前发送过来的消息为异步消息，判断该异步消息是否插入在所有异步消息的队首，如果是则需要唤醒，如果不是，则不唤醒

### 11、线程什么情况下会被阻塞？

答：分情况

1、当MessageQueue中没有消息的时候，这个时候会无限阻塞，

2、当前MessageQueue中全部是延迟消息，阻塞时间为(当前延迟消息时间 - 当前时间)，如果这个阻塞时间超过来Integer类型的最大值，则取Integer类型的最大值





### 常见面试

![image-20210524184449133](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210630180039.png)



# Message

## 14、我们在使用Message的时候，应该如何去创建它？

答：Android 给 Message 设计了回收机制，官方建议是通过`Message.obtain`方法来获取，而不是直接new一个新的对象，所以我们在使用的时候应尽量复用 Message ，减少内存消耗，方式有二：

1、调用 Message 的一系列静态重载方法 `Message.obtain`  获取

2、通过 Handler 的公有方法 `handler.obtainMessage`，实际上`handler.obtainMessage`内部调用的也是`Message.obtain`的重载方法



## Message#obtain实现细节了解么？为何要池化？最大限制容量是多少？

https://www.jianshu.com/p/e271ee639b68



https://droidyue.com/blog/2016/12/12/dive-into-object-pool/

https://www.jianshu.com/p/cb81031814f0

https://cloud.tencent.com/developer/article/1199403

https://toutiao.io/posts/ptdi4q/preview

https://www.dazhuanlan.com/2019/12/24/5e02022da1e64/

https://guolei1130.github.io/2017/02/15/%E4%BA%86%E8%A7%A3%E5%AF%B9%E8%B1%A1%E6%B1%A0/



## Message消息被分发之后会怎么处理？消息怎么复用的？

再看看loop方法，在消息被分发之后，也就是执行了`dispatchMessage`方法之后，还偷偷做了一个操作——`recycleUnchecked`。



```java
    public static void loop() {
        for (;;) {
            Message msg = queue.next(); // might block

            try {
                msg.target.dispatchMessage(msg);
            } 

            msg.recycleUnchecked();
        }
    }

//Message.java
    private static Message sPool;
    private static final int MAX_POOL_SIZE = 50;

    void recycleUnchecked() {
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
复制代码
```

在`recycleUnchecked`方法中，释放了所有资源，然后将当前的空消息插入到sPool表头。

这里的`sPool`就是一个消息对象池，它也是一个链表结构的消息，最大长度为50。

那么Message又是怎么复用的呢？在Message的实例化方法`obtain`中：



```java
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
复制代码
```

直接复用消息池`sPool`中的第一条消息，然后sPool指向下一个节点，消息池数量减一。



## Message是怎么找到它所属的Handler然后进行分发的？

在loop方法中，找到要处理的`Message`，然后调用了这么一句代码处理消息：



```css
msg.target.dispatchMessage(msg);
复制代码
```

所以是将消息交给了`msg.target`来处理，那么这个target是啥呢？

找找它的来头：



```cpp
//Handler
    private boolean enqueueMessage(MessageQueue queue,Message msg,long uptimeMillis) {
        msg.target = this;

        return queue.enqueueMessage(msg, uptimeMillis);
    }
复制代码
```

在使用Hanlder发送消息的时候，会设置`msg.target = this`，所以target就是当初把消息加到消息队列的那个Handler。



## Message是怎么创建的？

Message.obtain

享元模式：

单独new Message会存在内存抖动：     new      释放   （内存时大时小，会频繁gc）——>会导致卡顿

![image-20210708154140476](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708154140.png)



源码：为什么使用了设计模式？设计模式怎么使用？

知道，明白，掌握(自己动手写)

写代码用到



# 本章主要讲线程通信原理相关面试问题，包括消息队列的创建，消息循环机制，消息延时，同步和异步消息，消息屏障等等内容。

-  8-1 线程的消息队列是怎么创建的？ (09:55)
-  8-2 说说android线程间消息传递机制 (14:54)
-  8-3 handler的消息延时是怎么实现的？ (10:41)
-  8-4 说说idleHandler的原理 (14:42)
-  8-5 主线程进入loop循环了为什么没有ANR？ (12:47)
-  8-6 听说过消息屏障么？ 