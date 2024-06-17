# Android中创建多进程[[0-进程#进程创建 0-进程保活 进程创建]]

**1） 第一种，大家熟知的，就是给四大组件再`AndroidManifest`中指定`android:process`属性。**

```
<activity android:name="com.example.uithread.UIActivity" 
      android:process=":test"/>

   <activity android:name="com.example.uithread.UIActivity2"
      android:process="com.example.test"/>
```

可以看到，`android:process`有两种表达方式：

- `：test`。“：”的含义是指要在当前的进程名前面加上当前的包名，如果当前包名为com.example.jimu。那么这个进程名就应该是com.example.jimu：test。这种冒号开头的进程属于当前应用的私有进程，其他应用的组件不可以和他跑到同一个进程中。
- `com.example.test`。第二种表达方式，是完整的命名方式，它就是新进程的进程名，这种属于全局进程，其他应用可以通过shareUID的方式跑到同一个进程中。

简单说下`shareUID`：正常来说，Android中每个app都是一个单独的进程，与之对应的是一个唯一的linux user ID，所以就能保住该应用程序的文件或者组件只对该应用程序可见。但是也有一个办法能让不同的apk进行共享文件，那就是通过shareUID，它可以使不同的apk使用相同的 user ID。贴下用法:

```
//app1
<manifest package="com.test.app1"
android:sharedUserId="com.test.jimu"
>

//app2
<manifest package="com.test.app2"
android:sharedUserId="com.test.jimu"
>

//app1中获取app2的上下文：
Context mContext=this.createPackageContext("com.test.app2", Context.CONTEXT_IGNORE_SECURITY);
```

**2）第二种创建进程的方法，就是通过JNI在`native`层中去fork一个进程。**

这种就比较复杂了，我在网上找了一些资料，找到一个fork普通进程的：

```
//主要代码
long add(long x,long y) {
   //fpid表示fork函数返回的值 
    pid_t fpid; 
    int count=0; 
    fpid=fork();  
}

//结果：
USER       PID   PPID   VSZ     RSS  STAT  NAME                 
root       152  1              S    zygote
u0_a66   17247  152   297120  44096  S  com.example.jni
u0_a66   17520  17247  0    0    Z  com.example.jni
```

最终的结果是可以创建出一个进程，但是没有运行，占用的内存为0，处于僵尸程序状态。

但是它这个是通过普通进程`fork`出来的，我们知道Android中所有的进程都是直接通过zygote进程fork出来的（fork可以理解为孵化出来的当前进程的一个副本）。所以不知道直接去操作zygote进程可不可以成功，有了解的小伙伴可以在微信讨论群里给大家说说。

对了，有的小伙伴可能会问，为什么所有进程都必须用`zygote`进程fork呢？

- 这是因为`fork`的行为是复制整个用户的空间数据以及所有的系统对象，并且只复制当前所在的线程到新的进程中。也就是说，父进程中的`其他线程`在子进程中都消失了，为了防止出现各种问题（比如死锁，状态不一致）呢，就只让`zygote`进程，这个单线程的进程，来fork新进程。
- 而且在`zygote`进程中会做好一些初始化工作，比如启动虚拟机，加载系统资源。这样子进程fork的时候也就能直接共享，提高效率，这也是这种机制的优点。

# 一个应用使用多进程会有什么问题吗？

上面说到创建进程的方法很简单，写个android:process属性即可，那么使用是不是也这么简单呢？很显然不是，一个应用中多进程会导致各种各样的问题，主要有如下几个：

- **静态成员和单例模式完全失效**。因为每个进程都会分配到一个独立的虚拟机，而不同的虚拟机在内存分配上有不同的地址空间，所以在不同的进程，也就是不同的虚拟机中访问同一个类的对象会产生多个副本。
- **线程同步机制完全失效**。同上面一样，不同的内存是无法保证线程同步的，因为线程锁的对象都不一样了。
- **SharedPreferences不在可靠**。之前有一篇说SharedPreferences的文章中说过这一点，SharedPreferences是不支持多进程的。
- **Application会多次创建**。多进程其实就对应了多应用，所以新进程创建的过程其实就是启动了一个新的应用，自然也会创建新的Application，Application和虚拟机和一个进程中的组件是一一对应的。

# Android中的IPC方式 [[0-进程#Android IPC 进程隔离 跨进程通信（ IPC ） 进程通信方式]]

既然多进程有很多问题，自然也就有解决的办法，虽然不能共享内存，但是可以进行数据交互啊，也就是可以进行多进程间通信，简称IPC。

下面就具体说说Android中的八大IPC方式：

**1. Bundle Android四大组件都是支持在Intent中使用`Bundle`来传递数据**，所以四大组件直接的进程间通信就可以使用Bundle。但是Bundle有个大小限制要注意下，`bundle`的数据传递限制大小为1M，如果你的数据超过这个大小就要使用其他的通信方式了。

**2. 文件共享 这种方式就是多个进程通过`读写一个文件`来交换数据，完成进程间通信。**但是这种方式有个很大的弊端就是多线程读写容易出问题，也就是`并发问题`，如果出现并发读或者并发写都容易出问题，所以这个方法适合对数据同步要求不高的进程直接进行通信。

这里可能有人就奇怪了，`SharedPreference`不就是读写xml文件吗？怎么就不支持进程间通信了？

- 这是因为系统对于`SharedPreference`有读写缓存策略，也就是在内存中有一份SharedPreference文件的缓存，涉及到内存了，那肯定在多进程中就不那么可靠了。

**3. Messenger`Messenger`是用来传递Message对象的，在Message中可以放入我们要传递的数据**。它是一种轻量级的IPC方案，底层实现是AIDL。

看看用法，客户端和服务端通信：

```
//客户端
public class MyActivity extends Activity {

    private Messenger mService;
    private Messenger mGetReplyMessager=new Messenger(new MessengerHandler());

    private static class MessengerHandler extends Handler{
        @Override
        public void handleMessage(@NonNull Message msg) {
            switch (msg.what){
                case 1:
                    //收到消息
                    break;
            }
            super.handleMessage(msg);
        }
    }

    private ServiceConnection mConnection=new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService = new Messenger(service);
            Message msg = Message.obtain(null,0);
            Bundle bundle = new Bundle();
            bundle.putString("test", "message1");
            msg.setData(bundle);
            msg.replyTo = mGetReplyMessager; //设置接受消息者
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Intent i=new Intent(this,ServerService.class);
        bindService(i,mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}

//服务端    
public class ServerService extends Service {

    Messenger messenger = new Messenger(new MessageHandler());

    public ServerService() {
    }

    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }

    private class MessageHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 0:
                    //收到消息并发送消息
                    Messenger replyTo = msg.replyTo;
                    Message replyMsg = Message.obtain(null,1);
                    Bundle bundle = new Bundle();
                    bundle.putString("test","111");
                    replyMsg.setData(bundle);
                    try {
                        replyTo.send(replyMsg);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
            }
            super.handleMessage(msg);
        }
    }

}
```

**4. AIDL**

Messenger虽然可以发送消息和接收消息，但是无法同时处理大量消息，并且无法跨进程方法。但是AIDL则可以做到，这里简单说下`AIDL`的使用流程：

服务端首先建立一个Service监听客户端的连接请求，然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中申明，最后在Service中实现这个`AIDL`接口。

客户端需要绑定这个服务端的Service，然后将服务端返回的`Binder`对象转换成AIDL接口的属性，然后就可以调用AIDL中的方法了。

**5. ContentProvider**

这个大家应很熟悉了，四大组件之一，专门用于不同应用间进行数据共享的。它的底层实现是通过Binder实现的。主要使用方法有两步：

1、声明Provider

```
<provider 
    android:authorities="com.test.lz"  //provider的唯一标识 
    android:name=".BookProdiver"
    android:permission="com.test.permission"
    android:process=":provider"/>
```

- android:authorities，唯一标识，一般用包名。外界在访问数据的时候都是通过uri来访问的，uri由四部分组成

```
content://com.test.lz/ table/ 100
|            |          |       |
固定字段   Authority   数据表名   数据ID
```

- android:permission，权限属性，还有`readPermission，writePermission`。如果provider声明了权限相关属性，那么其他应用也必须声明相应的权限才能进行读写操作。比如：

```
//应用1
<permission android:name="me.test.permission" android:protectionLevel="normal"/>

 <provider 
    android:authorities="com.test.lz"
    android:name=".BookProdiver"
    android:permission="com.test.permission"
    android:process=":provider"/>  

//应用2
<uses-permission android:name="me.test.permission"/>
```

2、然后外界应用通过getContentResolver()的增删查改方法访问数据即可。

**6. Socket**

套接字，在网络通信中用的很多，比如TCP，UDP。关于Socket通信，借用网络上的一张图说明：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210721100446.png)

然后简单贴下关键代码:

```
//申请权限
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

//服务端
ServerSocket serverSocket = new ServerSocket(8688);
while(isActive) { 
        try {
         //不断获取客户端连接
              final Socket client = serverSocket.accept();
              new Thread(){
                        @Override
                        public void run() {
                            try {
                               //处理消息
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    }.start();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

//客户端
Socket socket = null;
    while(socket==null){  
        //为空代表链接失败，重连
        try {
            socket = new Socket("localhost",8688);
            out = new PrintWriter(socket.getOutputStream(),true);
            handler.sendEmptyMessage(1);
            final Socket finalSocket = socket;
            new Thread(){
                @Override
                public void run() {
                    try {
                        reader = new BufferedReader(new InputStreamReader(finalSocket.getInputStream()));
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    while(!MainActivity.this.isFinishing()){ 
                        //循环读取消息
                        try {
                            String msg = reader.readLine();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }.start();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

**7. Binder连接池**

关于Binder的介绍，之前的文章已经说过了。这里主要讲一个Binder的实际使用的技术——`Binder连接池`。由于每个AIDL请求都要开启一个服务，防止太多服务被创建，就引用了`Binder连接池`技术。Binder连接池的主要作用就是将每个业务模块的`Binder`请求统一 转发到远程Service中去执行，从而避免了重复创建`Service`的过程。贴一下Binder连接池的工作原理：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210721100459.png)

- 每个业务模块创建自己的AIDL接口并实现此接口，然后向服务端提供自己的唯一标识和其对应的Binder对象.
- 对于服务端来说，只需要一个 Service就可以了，服务端提供一个queryBinder接口，这个接口能够根据业务模块的特征来 返回相应的Binder对象给它们，不同的业务模块拿到所需的Binder对象后就可以进行远程方法调用了。

具体怎么用呢？还是简单贴下关键源码

```
public class BinderPoolImpl extends IBinderPool.Stub {
    private static final String TAG = "Service BinderPoolImpl";

    public BinderPoolImpl() {
        super();
    }

    @Override
    public IBinder queryBinder(int binderCode) throws RemoteException {
        Log.e(TAG, "binderCode = " + binderCode);
        IBinder binder = null;
        switch (binderCode) {
            case 0:
                binder = new SecurityCenterImpl();
                break;
            case 1:
                binder = new ComputeImpl();
                break;
            default:
                break;
        }
        return binder;
    }
}

public class BinderPool {
    //...
    private IBinderPool mBinderPool;

    private synchronized void connectBinderPoolService() {
        Intent service = new Intent();
        service.setComponent(new ComponentName("com.test.lz", "com.test.lz.BinderPoolService"));
        mContext.bindService(service, mBinderPoolConnection, Context.BIND_AUTO_CREATE);
    }    

    public IBinder queryBinder(int binderCode) {
        IBinder binder = null;
        try {
            if (mBinderPool != null) {
                binder = mBinderPool.queryBinder(binderCode);
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return binder;
    }

    private ServiceConnection mBinderPoolConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mBinderPool = IBinderPool.Stub.asInterface(service);
            try {
                mBinderPool.asBinder().linkToDeath(mBinderPoolDeathRecipient, 0);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    private IBinder.DeathRecipient mBinderPoolDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            Log.e(TAG, "binderDied");
            mBinderPool.asBinder().unlinkToDeath(mBinderPoolDeathRecipient, 0);
            mBinderPool = null;
            connectBinderPoolService();
        }
    };

}
```

**8. BroadcastReceiver**

广播，不用多说了吧~ 像我们可以监听系统的开机广播，网络变动广播等等，都是体现了进程间通信的作用。

# 总结

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210721100507.png)

面试前做好准备战！







# 参考

https://maimai.cn/article/detail?fid=1604382370&efid=cvXk4MpDHoKzetcXFssINQ&share_channel=2&use_rn=1&_share_channel=wechat



