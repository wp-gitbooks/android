# 内部类会造成程序的内存泄漏
要想了解为啥内部类为什么会造成内存泄漏我们就必须了解 java 虚拟机的回收机制，但是我们这里不会详尽的介绍 java 的内存回收机制，我们只需要了解 java 的内存回收机制通过「可达性分析」来实现的。即 java 虚拟机会通过内存回收机制来判定引用是否可达，如果不可达就会在某些时刻去回收这些引用。

那么内部类在什么情况下会造成内存泄漏的可能呢？

> 1. 如果一个匿名内部类没有被任何引用持有，那么匿名内部类对象用完就有机会被回收。
> 2. 如果内部类仅仅只是在外部类中被引用，当外部类的不再被引用时，外部类和内部类就可以都被GC回收。
> 3. 如果当内部类的引用被外部类以外的其他类引用时，就会造成内部类和外部类无法被GC回收的情况，即使**外部类没有被引用，因为内部类持有指向外部类的引用**）。

```java
public class ClassOuter {

    Object object = new Object() {
        public void finalize() {
            System.out.println("inner Free the occupied memory...");
        }
    };

    public void finalize() {
        System.out.println("Outer Free the occupied memory...");
    }
}

public class TestInnerClass {
    public static void main(String[] args) {
        try {
            Test();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static void Test() throws InterruptedException {
        System.out.println("Start of program.");

        ClassOuter outer = new ClassOuter();
        Object object = outer.object;
        outer = null;

        System.out.println("Execute GC");
        System.gc();

        Thread.sleep(3000);
        System.out.println("End of program.");
    }
}
```

运行程序发现 执行内存回收并没回收 object 对象，这是因为即使外部类没有被任何变量引用，只要其内部类被外部类以外的变量持有，外部类就不会被GC回收。我们要尤其注意内部类被外面其他类引用的情况，这点导致外部类无法被释放，极容易导致内存泄漏。


# Handler内存泄漏
## 例子&引用链关系
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

**当前线程->ThreadLocal->Looper->MessageQueue->Message->Handler->Activity**

**主线程 —> threadlocal —> Looper —> MessageQueue —> Message —> Handler —> Activity

因此当我们退出Activity的时候，由于消息需要在10分钟后在执行，因此会一直持有Activity，从而导致了Activity的内存泄漏

## 泄露原因
在Android 中 Hanlder 作为内部类使用的时候其对象被系统主线程的 Looper 持有（当然这里也可是子线程手动创建的 Looper）掌管的消息队列 MessageQueue 中的 Hanlder 发送的 Message 持有，当消息队列中有大量消息处理的需要处理，或者延迟消息需要执行的时候，创建该 Handler 的 Activity 已经退出了，Activity 对象也无法被释放，这就造成了内存泄漏。

那么 Hanlder 何时会被释放，当消息队列处理完 Hanlder 携带的 message 的时候就会调用 msg.recycleUnchecked()释放Message所持有的Handler引用。

## 解决
### 取消排队的Message
- 在关闭Activity/Fragment 的 onDestry，**取消还在排队的Message**:

```java
mHandler.removeCallbacksAndMessages(null);
```

### 静态内部类并采用弱引用
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


# 参考
[[3-内部类#内部类会造成程序的内存泄漏]]
[[Handler面试题#8、Handler内存泄露原因 如何解决？]]
[[3-Handler应用#26、说说Hanlder内存泄露问题。]]