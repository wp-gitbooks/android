# 线索
总览：通过一个例子
参数：类型、关键(in、out、inout)、oneway
原理

其他：
线程安全、死亡监听、权限验证


# 1 例子

## 1.1 Server端

RemoteService.java

本例是为了演示进程间的通信机制，故需要将Service与Activity处于不同的进程，需要在AndroidManifest.xml中，把service配置成`android:process=":remote"`,进程也可以命名成其他的。

```java
public class RemoteService extends Service {
    private static final String TAG = "BinderSimple";

    MyData mMyData;

    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(TAG, "[RemoteService] onCreate");
        initMyData();
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.i(TAG,"[RemoteService] onBind");
        return mBinder;
    }

    @Override
    public boolean onUnbind(Intent intent) {
        Log.i(TAG, "[RemoteService] onUnbind");
        return super.onUnbind(intent);
    }

    @Override
    public void onDestroy() {
        Log.i(TAG, "[RemoteService] onDestroy");
        super.onDestroy();
    }

    /** * 实现IRemoteService.aidl中定义的方法 */
    private final IRemoteService.Stub mBinder = new IRemoteService.Stub() {

        @Override
        public int getPid() throws RemoteException {
            Log.i(TAG,"[RemoteService] getPid()="+android.os.Process.myPid());
            return android.os.Process.myPid();
        }

        @Override
        public MyData getMyData() throws RemoteException {
            Log.i(TAG,"[RemoteService] getMyData() "+ mMyData.toString());
            return mMyData;
        }

        /**此处可用于权限拦截**/
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            return super.onTransact(code, data, reply, flags);
        }
    };

    /** * 初始化MyData数据 **/
    private void initMyData() {
        mMyData = new MyData();
        mMyData.setData1(10);
        mMyData.setData2(20);
    }
}
```

## 1.2 Client端

ClientActivity.java

```Java
public class ClientActivity extends AppCompatActivity {
    private static final String TAG = "BinderSimple";

    private IRemoteService mRemoteService;

    private boolean mIsBound;
    private TextView mCallBackTv;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Log.i(TAG, "[ClientActivity] onCreate");

        setContentView(R.layout.activity_main);

        mCallBackTv = (TextView) findViewById(R.id.tv_callback);
        mCallBackTv.setText(R.string.remote_service_unattached);
    }

    /**
     * 用语监控远程服务连接的状态
     */
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {

            mRemoteService = IRemoteService.Stub.asInterface(service);
            String pidInfo = null;
            try {
                MyData myData = mRemoteService.getMyData();
                pidInfo = "pid="+ mRemoteService.getPid() +
                        ", data1 = "+ myData.getData1() +
                        ", data2="+ myData.getData2();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            Log.i(TAG, "[ClientActivity] onServiceConnected  "+pidInfo);
            mCallBackTv.setText(pidInfo);
            Toast.makeText(ClientActivity.this, R.string.remote_service_connected, Toast.LENGTH_SHORT).show();
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.i(TAG, "[ClientActivity] onServiceDisconnected");
            mCallBackTv.setText(R.string.remote_service_disconnected);
            mRemoteService = null;
            Toast.makeText(ClientActivity.this, R.string.remote_service_disconnected, Toast.LENGTH_SHORT).show();
        }
    };


    public void clickHandler(View view){
        switch (view.getId()){
            case R.id.btn_bind:
                bindRemoteService();
                break;

            case R.id.btn_unbind:
                unbindRemoteService();
                break;

            case R.id.btn_kill:
                killRemoteService();
                break;
        }
    }

    /**
     * 绑定远程服务
     */
    private void bindRemoteService(){
        Log.i(TAG, "[ClientActivity] bindRemoteService");
        Intent intent = new Intent(ClientActivity.this, RemoteService.class);
        intent.setAction(IRemoteService.class.getName());
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);

        mIsBound = true;
        mCallBackTv.setText(R.string.binding);
    }

    /**
     * 解除绑定远程服务
     */
    private void unbindRemoteService(){
        if(!mIsBound){
            return;
        }
        Log.i(TAG, "[ClientActivity] unbindRemoteService ==>");
        unbindService(mConnection);
        mIsBound = false;
        mCallBackTv.setText(R.string.unbinding);
    }

    /**
     * 杀死远程服务
     */
    private void killRemoteService(){
        Log.i(TAG, "[ClientActivity] killRemoteService");
        try {
            android.os.Process.killProcess(mRemoteService.getPid());
            mCallBackTv.setText(R.string.kill_success);
        } catch (RemoteException e) {
            e.printStackTrace();
            Toast.makeText(ClientActivity.this, R.string.kill_failure, Toast.LENGTH_SHORT).show();
        }
    }
}
```

## 1.3 AIDL文件

(1)IRemoteService.aidl 定义远程通信的接口方法

```java
interface IRemoteService {
    int getPid();
    MyData getMyData();
}
```

(2)MyData.aidl 定义远程通信的自定义数据

```java
parcelable MyData;
```

## 1.4 Parcel数据
[[Parcelable和Serializable-序列化]]

MyData.java

```Java
public class MyData implements Parcelable {
    private int data1;
    private int data2;

    public MyData(){

    }

    protected MyData(Parcel in) {
        readFromParcel(in);
    }

    public static final Creator<MyData> CREATOR = new Creator<MyData>() {
        @Override
        public MyData createFromParcel(Parcel in) {
            return new MyData(in);
        }

        @Override
        public MyData[] newArray(int size) {
            return new MyData[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    /** 将数据写入到Parcel **/
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(data1);
        dest.writeInt(data2);
    }

    /** 从Parcel中读取数据 **/
    public void readFromParcel(Parcel in){
        data1 = in.readInt();
        data2 = in.readInt();
    }


    public int getData2() {
        return data2;
    }

    public void setData2(int data2) {
        this.data2 = data2;
    }

    public int getData1() {
        return data1;
    }

    public void setData1(int data1) {
        this.data1 = data1;
    }

    @Override
    public String toString() {
        return "data1 = "+ data1 + ", data2="+ data2;
    }
}
```



## 1.5 运行

该工程会生成一个apk，安装到手机，打开apk，界面如下：

![apk](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210723102450.png)

界面上有三个按钮，分别是功能分别是bindService(绑定Service), unbindService(解除绑定Service), killProcess(杀死Service进程)。

从左往右，依次点击界面，可得：

![apk](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210723102535.png)



# 参数类型 [[深入AIDL#2 3 in out inout关键字]]
|All non-primitive parameters require a directional tag indicating which way the data goes. Either in, out, or inout. Primitives are in by default, and cannot be otherwise.  
**Caution:** You should limit the direction to what is truly needed, because marshalling parameters is expensive.

  
大致的意思是说所有非基本数据类型的参数在传递的时候都需要指定一个方向tag来指明数据的流向，可以是in、out或者inout。基本数据类型默认是in，还不能修改为其他的。

## 基本数据类型

我们都知道Java有8中基本数据类型，分别为byte,char,short,int,long,float,double,boolean,那这8中数据类型是否都能作为aidl的参数进行传递呢？我们可以在aild中尝试下，并编译，看看有没有错

```
void basicTypes(byte aByte, char aChar, short aShort, int anInt, long aLong, float aFloat,
            double aDouble, boolean aBoolean);
复制代码
```

发现这样子定义，无法成功编译，经过筛查发现aidl并不能支持short基本数据类型，至于为什么呢，可以看一看Android中的Parcel，Parcel是不支持short的，这应该是考虑到兼容性问题吧。

**所以基本数据类型支持：byte,char,int,long,float,double,boolean**

## 引用数据类型

引用数据类型根据[官方]("https://developer.android.google.cn/guide/components/aidl.html")介绍，可以使用String,CharSequence,List,Map，当然，我们也可以使用自定义数据类型。

## 自定义数据类型

自定义数据类型，用于进程间通信的话，必须实现Parcelable接口，Parcelable是类似于Java中的Serializable，Android中定义了Parcelable，用于进程间数据传递，对传输数据进行分解，编组的工作，相对于Serializable，他对于进程间通信更加高效。

我们来看下下面的例子

```
public class User implements Parcelable {

    private int id;
    private String name;

    public User() {
    }

    public User(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public User(Parcel in){
      //注意顺序！！！注意顺序！！！注意顺序！！！
        this.id = in.readInt();
        this.name = in.readString();
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
      //注意顺序！！！注意顺序！！！注意顺序！！！
        dest.writeInt(id);
        dest.writeString(name);
    }

    public static final  Parcelable.Creator<User> CREATOR = new Parcelable.Creator<User>(){

        @Override
        public User createFromParcel(Parcel source) {
            return new User(source);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
复制代码
```

这边我们定义了一个User类，实现了Parcelable接口，大致的类结构就是这个样子的，需要注意的一点是，Parcelable对数据进行分解/编组的时候必须使用相同的顺序，字段以什么顺序分解的，编组时就以什么顺序读取数据，不然会有问题！

## in、out和inout

http://dandanlove.com/2016/10/27/aidl-study/

https://dandanlove.com/2016/10/27/aidl-tag-type/

https://blog.csdn.net/anlian523/article/details/98476033

https://www.jianshu.com/p/a61da801b919

https://www.dtmao.cc/news_show_230185.shtml

https://zhuanlan.zhihu.com/p/338093696

-   in表示输入型参数（Server可以获取到Client传递过去的数据，但是不能对Client端的数据进行修改）
-   out表示输出型参数（Server获取不到Client传递过去的数据，但是能对Client端的数据进行修改）
-   inout表示输入输出型参数（Server可以获取到Client传递过去的数据，但是能对Client端的数据进行修改）。



# 2 原理分析
[[1-Android插件化开发指南#AIDL原理]]

## 调用图
![aidl image](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210723102240.jpg)
![image-20210723104911593](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210723104911.png)
## 类关系图
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210729145612.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210729150121.png)

### 获取代理对象Proxy

```java
IRemoteService.Stub.asInterface(service);
```

### 实现接口
```
    /** * 实现IRemoteService.aidl中定义的方法 */
    private final IRemoteService.Stub mBinder = new IRemoteService.Stub() {

        @Override
        public int getPid() throws RemoteException {
            Log.i(TAG,"[RemoteService] getPid()="+android.os.Process.myPid());
            return android.os.Process.myPid();
        }

        @Override
        public MyData getMyData() throws RemoteException {
            Log.i(TAG,"[RemoteService] getMyData() "+ mMyData.toString());
            return mMyData;
        }

        /**此处可用于权限拦截**/
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            return super.onTransact(code, data, reply, flags);
        }
    };
```


## 数据流  [[0-Binder#通信协议]]
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210729151505.png)



## 源码

采用AIDL技术，是原理还是利用framework binder的架构。本文的实例AIDL会自动生成一个与之相对应的IRemoteService.java文件，如下：

```Java
package com.gityuan.appbinderdemo;
public interface IRemoteService extends android.os.IInterface {
    public static abstract class Stub extends android.os.Binder implements com.gityuan.appbinderdemo.IRemoteService {
        private static final java.lang.String DESCRIPTOR = "com.gityuan.appbinderdemo.IRemoteService";

        /**
         * Stub构造函数
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * 将IBinder 转换为IRemoteService interface
         */
        public static com.gityuan.appbinderdemo.IRemoteService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.gityuan.appbinderdemo.IRemoteService))) {
                return ((com.gityuan.appbinderdemo.IRemoteService) iin);
            }
            return new com.gityuan.appbinderdemo.IRemoteService.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                
                case TRANSACTION_getPid: {
                    data.enforceInterface(DESCRIPTOR);
                    int _result = this.getPid();
                    reply.writeNoException();
                    reply.writeInt(_result);
                    return true;
                }
                case TRANSACTION_getMyData: {
                    data.enforceInterface(DESCRIPTOR);
                    com.gityuan.appbinderdemo.MyData _result = this.getMyData();
                    reply.writeNoException();
                    if ((_result != null)) {
                        reply.writeInt(1);
                        _result.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                    } else {
                        reply.writeInt(0);
                    }
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.gityuan.appbinderdemo.IRemoteService {
            private android.os.IBinder mRemote;

            /**
             * Proxy构造函数
             */
            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public int getPid() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                int _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getPid, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readInt();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public com.gityuan.appbinderdemo.MyData getMyData() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                com.gityuan.appbinderdemo.MyData _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getMyData, _data, _reply, 0);
                    _reply.readException();
                    if ((0 != _reply.readInt())) {
                        _result = com.gityuan.appbinderdemo.MyData.CREATOR.createFromParcel(_reply);
                    } else {
                        _result = null;
                    }
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_getPid = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getMyData = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public int getPid() throws android.os.RemoteException;

    public com.gityuan.appbinderdemo.MyData getMyData() throws android.os.RemoteException;
}
```


# 深入AIDL [[深入AIDL]]



# 应用 [[3-PMS#概述]]


# 参考
https://developer.android.com/guide/components/aidl?hl=zh-cn

http://weishu.me/2016/01/12/binder-index-for-newer/


http://gityuan.com/2015/11/23/binder-aidl/




