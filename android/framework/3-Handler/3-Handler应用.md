# Handler应用之HandlerThread
HandlerThread同Looper关联起来



![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210330222412)

## 使用

```java
// 步骤1：创建HandlerThread实例对象
// 传入参数 = 线程名字，作用 = 标记该线程
   HandlerThread mHandlerThread = new HandlerThread("handlerThread");

// 步骤2：启动线程
   mHandlerThread.start();

// 步骤3：创建工作线程Handler & 复写handleMessage（）
// 作用：关联HandlerThread的Looper对象、实现消息处理操作 & 与其他线程进行通信
// 注：消息处理操作（HandlerMessage（））的执行线程 = mHandlerThread所创建的工作线程中执行
  Handler workHandler = new Handler( mHandlerThread.getLooper() ) {
            @Override
            public boolean handleMessage(Message msg) {
                ...//消息处理
                return true;
            }
        });

// 步骤4：使用工作线程Handler向工作线程的消息队列发送消息
// 在工作线程中，当消息循环时取出对应消息 & 在工作线程执行相关操作
  // a. 定义要发送的消息
  Message msg = Message.obtain();
  msg.what = 2; //消息的标识
  msg.obj = "B"; // 消息的存放
  // b. 通过Handler发送消息到其绑定的消息队列
  workHandler.sendMessage(msg);

// 步骤5：结束线程，即停止线程的消息循环
  mHandlerThread.quit();
```



## 源码

```java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}
```


# Handler应用之IntentService
IntentService是Service的子类,由于Service里面不能做耗时的操作,所以Google提供了IntentService,在**IntentService内维护了一个工作线程来处理耗时操作**，当任务执行完后，IntentService会自动停止。另外，可以启动IntentService多次，而每一个耗时操作会以工作队列的方式在IntentService的**onHandleIntent**回调方法中执行，并且，每次只会执行一个工作线程，执行完第一个再执行第二个，以此类推。

## 使用
```java
public class MyService extends IntentService {
    //这里必须有一个空参数的构造实现父类的构造,否则会报异常
    //java.lang.InstantiationException: java.lang.Class<***.MyService> has no zero argument constructor
    public MyService() {
        super("");
    }
    
    @Override
    public void onCreate() {
        System.out.println("onCreate");
        super.onCreate();
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        System.out.println("onStartCommand");
        return super.onStartCommand(intent, flags, startId);

    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        System.out.println("onStart");
        super.onStart(intent, startId);
    }

    @Override
    public void onDestroy() {
        System.out.println("onDestroy");
        super.onDestroy();
    }

    //这个是IntentService的核心方法,它是通过串行来处理任务的,也就是一个一个来处理
    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        System.out.println("工作线程是: "+Thread.currentThread().getName());
        String task = intent.getStringExtra("task");
        System.out.println("任务是 :"+task);
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 源码

### IntentService
```java
@Override
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.
 
    super.onCreate();
    // 创建HandlerThread
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();
 
    // 创建Handler和HandlerThread绑定
    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
 
@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    // 想HandlerThread的消息队列发送消息
    mServiceHandler.sendMessage(msg);
}
```



### ServiceHandler

```
private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }
 
        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

```
public ServiceHandler(Looper looper) {
        super(looper);
    }
 
    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        // 答案在这里：在这里会停止Service
        stopSelf(msg.arg1);
    }
}
 
// 然后在onDestory里会终止掉消息循环，从而达到销毁异步线程的目的：
@Override
public void onDestroy() {
    mServiceLooper.quit();
}
```



# Handler.post和View.post

https://segmentfault.com/a/1190000038350183



# Handler(实战)

http://gityuan.com/2016/01/01/handler-message-usage/



# 面试题



## 23、HandlerThread是啥？有什么使用场景？

直接看源码：

```java
public class HandlerThread extends Thread {
    @Override
    public void run() {
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
    }
复制代码
```

哦，原来如此。`HandlerThread`就是一个封装了Looper的Thread类。

就是为了让我们在子线程里面更方便的使用Handler。

这里的加锁就是为了保证线程安全，获取当前线程的Looper对象，获取成功之后再通过`notifyAll`方法唤醒其他线程，那哪里调用了`wait`方法呢？



```kotlin
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }

        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
复制代码
```

就是`getLooper`方法，所以wait的意思就是等待Looper创建好，那边创建好之后再通知这边正确返回Looper。

## 24、IntentService是啥？有什么使用场景？

老规矩，直接看源码：

```java
public abstract class IntentService extends Service {

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

```

理一下这个源码：

- 首先，这是一个`Service`
- 并且内部维护了一个`HandlerThread`，也就是有完整的Looper在运行。
- 还维护了一个子线程的`ServiceHandler`。
- 启动Service后，会通过Handler执行`onHandleIntent`方法。
- 完成任务后，会自动执行`stopSelf`停止当前Service。

所以，这就是一个可以在子线程进行耗时任务，并且在任务执行后自动停止的Service。

## 25、BlockCanary使用过吗？说说原理

`BlockCanary`是一个用来检测应用卡顿耗时的三方库。

上文说过，View的绘制也是通过Handler来执行的，所以如果能知道每次Handler处理消息的时间，就能知道每次绘制的耗时了？ 那Handler消息的处理时间怎么获取呢？

再去loop方法中找找细节：



```java
public static void loop() {
    for (;;) {
        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
    }
}

```

可以发现，loop方法内有一个`Printer`类，在`dispatchMessage`处理消息的前后分别打印了两次日志。

那我们把这个日志类`Printer`替换成我们自己的`Printer`，然后统计两次打印日志的时间不就相当于处理消息的时间了？


```java
    Looper.getMainLooper().setMessageLogging(mainLooperPrinter);

    public void setMessageLogging(@Nullable Printer printer) {
        mLogging = printer;
    }

```

这就是BlockCanary的原理。

## 26、说说Hanlder内存泄露问题。

这也是常常被问的一个问题，`Handler`内存泄露的原因是什么？

**"内部类持有了外部类的引用，也就是Hanlder持有了Activity的引用，从而导致无法被回收呗。"**

其实这样回答是错误的，或者说没回答到点子上。

我们必须找到那个最终的引用者，不会被回收的引用者，其实就是主线程，这条完整引用链应该是这样：

**主线程 —> threadlocal —> Looper —> MessageQueue —> Message —> Handler —> Activity**

## 27、利用Handler机制设计一个不崩溃的App？
主线程崩溃，其实都是发生在消息的处理内，包括生命周期、界面绘制。
所以如果我们能控制这个过程，并且在发生崩溃后重新开启消息循环，那么主线程就能继续运行。

```java
Handler(Looper.getMainLooper()).post {
        while (true) {
            //主线程异常拦截
            try {
                Looper.loop()
            } catch (e: Throwable) {
            }
        }
    }

```

## 总结
大家应该可以发现，有一个问题常被问，但是全篇都没有提，那就是：
Hanlder机制的运行原理。


## HandlerThread
上面讲到主线程发送消息到子线程来处理，其实Android已经给我们提供了一个这样轻量级的异步类，那就是HandlerThread

HandlerThread的实现原理也比较简单：**继承Thread并对Looper进行了封装**

具体源码就不过多分析了，大家有兴趣的可以去看一下，也就100多行代码，这里主要讲解一下使用：



```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //1，创建Handler实例
        HandlerThread mHandlerThread = new HandlerThread("HandlerThread");
        //2，启动线程
        mHandlerThread.start();
        //3，使用传入Looper为参数的构造方法创建Handler实例
        Handler mHandler = new Handler(mHandlerThread.getLooper()){
            @Override
            public void handleMessage(@NonNull Message msg) {
                Log.d("print", "当前线程: " + Thread.currentThread().getName() + " handleMessage");
            }
        };
        //4，使用Handler发送消息
        mHandler.sendEmptyMessage(0x001);
        //5，在合适的时机调用HandlerThread的quit方法，退出消息循环
    }
}

//上述代码打印结果:
当前线程: HandlerThread handleMessage
```

## Handler  HandlerThread  Thread三者区别
**Handler**：在Android中负责发送和处理消息

**HandlerThread**：继承自Thread，对Looper进行了封装，也就是说它在子线程维护了一个Looper，方便我们在子线程中去处理消息

**Thread**: cpu执行的最小单位，即线程，它在执行完后就立马结束了，并不能去处理消息。如果要处理，需要配合Looper，Handler一起使用

## 子线程弹Toast
```java
//1
new Thread(){
    @Override
    public void run() {
        Toast.makeText(MainActivity.this, "子线程弹Toast", Toast.LENGTH_SHORT).show();
    }
}.start();

//2
new Thread(){
    @Override
    public void run() {
        Looper.prepare();
        Toast.makeText(MainActivity.this, "子线程弹Toast", Toast.LENGTH_SHORT).show();
        Looper.loop();
    }
}.start();
```

上述1代码运行会奔溃，会报这么一个异常提示：**"Can't toast on a thread that has not called Looper.prepare()"**

原因就是Toast的实现也是依赖Handler，而我们知道在子线程中创建Handler，需先创建Looper并开启消息循环，这点在Toast中的源码也有体现，如下图：

![img](https:////upload-images.jianshu.io/upload_images/4311670-c327d383cc87aca3.image?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

因此我们在子线程创建Toast就需要使用上述2代码的方式

## 子线程弹Dialog

```java
new Thread(){
    @Override
    public void run() {
            Looper.prepare();
            new AlertDialog.Builder(MainActivity.this)
                  .setTitle("标题")
                  .setMessage("子线程弹Dialog")
                  .setNegativeButton("取消",null)
                  .setPositiveButton("确定",null)
                  .show();
            Looper.loop();     
    }    
}.start();
```

和上面Toast差不多，这里贴出正确的代码示例，它的实现也是依赖Handler，我们在它的源码中可以看到：

```java
private final Handler mHandler = new Handler();
```

他直接就new了一个Handler实例，我们知道，创建Handler，需要先创建Looper并开启消息循环，主线程中已经给我们创建并开启消息循环，而子线程中并没有，如果不创建那就会报这句经典的异常提示：**"Can't create handler inside thread that has not called Looper.prepare() "**，因此在子线程中，需要我们手动去创建并开启消息循环

到这里，Handler相关的扩展知识就全部讲完了，我们会发现也有着很多使用的小技巧，比如 IdleHandler，判断是否是主线程等等

由于 Handler 的特性，它在 Android 里的应用非常广泛，比如： AsyncTask、HandlerThread、Messenger、IdleHandler 和 IntentService 等等，下面我们来回答上一篇中列出来的一系列问题


# 参考

https://www.jianshu.com/p/4a58568661e8

https://www.jianshu.com/p/f91cb08f8b7c
