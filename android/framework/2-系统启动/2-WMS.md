---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 线索


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303101140314.png)



Activity、Window、DecorView、View的绘制流程


# 2 WMS&AMS
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303071443202.png)

AMS 和 WMS 什么关系？
都属于同一个进程system_server进程，都是在手机启动system_server的时候启动


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303071517835.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303082118038.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303082113686.png)

application不能创建 window

Dialog 是什么类型window

黑白屏window
启动黑白屏解决？


问题4：
https://blog.csdn.net/binghelonglong123/article/details/95333516
https://juejin.cn/post/6844904192826425357vv



问题5：是同一个
https://www.bilibili.com/video/av256486209/?vd_source=f089cad7f2e400622e91946932061cb9
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303082127798.png)




# 3 Framework之WMS解析
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303082005724.png)



![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303082013371.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303082014207.png)

问题一：
textview.invalidate():会触发其他view 的绘制。主要原理会从下到上，计算受影响的view区域和标记，找到 ViewRootImpl，然后从上到下开始绘制受影响的view。
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303082034853.png)

问题2：


# 4 Window加载视图过程
## 4.1 整体视图关系
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081533261.png)

**Activity**: 本身不是窗口，也不是视图，它只是窗口的载体，一方面是key、touch事件的响应处理，一方面是对界面生命周期做统一调度

**Window**: 一个顶级窗口查看和行为的一个抽象基类。它也是装载View的原始容器， 处理一些应用窗口通用的逻辑。使用时用到的是它的唯一实现类：PhoneWindow。Window受WindowManager统一管理。

**DecorView**: 顶级视图，一般情况下它内部会包含一个竖直方向的LinearLayout，上面的标题栏(titleBar)，下面是内容栏。通常我们在Activity中通过setContentView所设置的布局文件就是被加载到id为android.R.id.content的内容栏里(FrameLayout)。

### 4.1.1 window的类型
https://www.jianshu.com/p/bac61386d9bf

添加窗口是通过WindowManagerGlobal的addView方法操作的，这里有三个必要参数：view，params，display。

**display** : 表示要输出的显示设备。

**view** : 表示要显示的View，一般是对该view的上下文进行操作。(view.getContext())

**params** : 类型为WindowManager.LayoutParams，即表示该View要展示在窗口上的布局参数。有2个比较重要的参数:`flags`,`type`。

**-flags:表示Window的属性:**
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081537860.png)

**-type: 表示窗口的类型（具体值太多了就不一一列举了，具体可以去WindowManager的LayoutParams看详细类型描述）:**

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081537184.png)


**这个type层级到底有什么作用：**  
Window是分层的，每个Window都有对应的z-ordered，（z轴，从1层层叠加到2999，你可以将屏幕想成三维坐标模式）层级大的会覆盖在层级小的Window上面。如果想要Window位于所有Window的最顶层，那么采用较大的层级即可。另外有些系统层级的使用是需要声明权限的。

那么照这么说，最底层的应该是应用window，子window在其上，系统window在最上面，这也符合视图展示的预期效果。

另外WindowManager的LayoutParams中还有个token必须要提一嘴：主要作用是为了维护activity和window的对应关系。

## 4.2 window的创建
讲window的创建过程，那么肯定得了解activity的启动流程，但是在这里不详细说activitiy的启动流程了，因为后面会有计划单独拎出四大组件开篇章来讲解启动流程。

那么简单的先上个图了解下activity的启动流程（借用辉辉的图）：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081538657.png)


对于window的创建，我们就从handleLaunchActivity开始，开始看源码吧：

```java
//ActivityThread
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ........
    //获取WindowManagerService的Binder引用(proxy端)。
    WindowManagerGlobal.initialize();
    //创建activity,调用attach方法，然后调用Activity的onCreate,onStart,onResotreInstanceState方法
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        ........
        //会调用Activity的onResume方法.
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
        .........
    }
}
```

主要看 Activity a = performLaunchActivity(r, customIntent);方法，关注Activity的attach方法：

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window) {
        //绑定上下文
        attachBaseContext(context);
        //创建Window, PhoneWindow是Window的唯一具体实现类
        mWindow = new PhoneWindow(this, window);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        ......
        //设置WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        //创建完后通过getWindowManager就可以得到WindowManager实例
        mWindowManager = mWindow.getWindowManager();//其实它是WindowManagerImpl
    }
```

这里创建了一个PhoneWindow对象，并且实现了Window的Callback接口，这样activity就和window关联在了一起，并且通过callback能够接受key和touch事件。

此外，初始化且设置windowManager。每个 Activity 会有一个 WindowManager 对象，这个 mWindowManager 就是和 WindowManagerService 进行通信，也是 WindowManagerService 识别 View 具体属于那个 Activity 的关键，创建时传入 IBinder 类型的 mToken。

```css
mWindow.setWindowManager(..., mToken, ..., ...)
```

我们从window的setWindowManager方法出发，很容易找到WindowManager这个接口的具体的实现是WindowManagerImpl。

### 4.2.1 时序图
```plantuml
ActivityThread -> ActivityThread:handleLaunchActivity
ActivityThread -> ActivityThread:performLaunchActivity
ActivityThread -> Instrumentation:newActivity
Instrumentation -> Activity:attach
Activity -> PhoneWindow:setWindowControllerCallback

```


### 4.2.2 Dialog的window创建
https://blog.csdn.net/weixin_38196407/article/details/106358950

## 4.3 window添加view过程
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081545463.png)
我们前面知道PhoneWindow对View来说更多是扮演容器的角色，而真正完成把一个 View，作为窗口添加到 WMS 的过程是由 WindowManager 来完成的。而且从上面创建过程我们知道了WindowManager 的具体实现是 WindowManagerImpl。

那么我们继续来跟代码：
### 4.3.1 ActivityThread
从上面handleLaunchActivity的代码中performLaunchActivity后面，有个handleResumeActivity，从名字也能看出，跟activity onResume相关。进去看看：

```java
//ActivityThread
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        //把activity数据记录更新到ActivityClientRecord
        ActivityClientRecord r = mActivities.get(token);
        r = performResumeActivity(token, clearHide, reason);
        if (r != null) {
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);//不可见
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
             .....
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);//把decor添加到窗口上（划重点）
                }
            }
                //屏幕参数发生了改变
                performConfigurationChanged(r.activity, r.tmpConfig);
                WindowManager.LayoutParams l = r.window.getAttributes();
                    if (r.activity.mVisibleFromClient) {
                        ViewManager wm = a.getWindowManager();
                        View decor = r.window.getDecorView();
                        wm.updateViewLayout(decor, l);//更新窗口状态
                    }
                ......
                if (r.activity.mVisibleFromClient) {
                    //已经成功添加到窗口上了（绘制和事件接收），设置为可见
                    r.activity.makeVisible();
                }
            //通知ActivityManagerService，Activity完成Resumed
             ActivityManagerNative.getDefault().activityResumed(token);
        }
    }
```

我们注意到这么几行代码：

```bash
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);/
                }
```

### 4.3.2 WindowManagerImpl
wm是activity getWindowManager()获取的，那不就是WindowManagerImpl的addView方法吗，追！

```java
//WindowManagerImpl
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

我们看到了又代理了一层：mGlobal， 它是谁？ 如果有印象的会记得讲window类型的时候带了一嘴的WindowManagerGlobal ，那这个WindowManagerImpl原来也是一个吃空饷的家伙！对于Window(或者可以说是View)的操作都是交由WindowManagerGlobal来处理，WindowManagerGlobal以工厂的形式向外提供自己的实例。这种工作模式是桥接模式，将所有的操作全部委托给WindowManagerGlobal来实现。

### 4.3.3 WindowManagerGlobal
讲setView之前先普及下WindowManager与WindowManagerService binder IPC的两个接口：

IWindowSession: 应用程序向WMS请求功能  
实现类：Session  
IWindow：WMS向客户端反馈它想确认的信息  
实现类：W

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081544228.png)



在WindowManagerImpl的全局变量中通过单例模式初始化了WindowManagerGlobal，也就是说一个进程就只有一个WindowManagerGlobal对象。那看看它：

```java
//WindowManagerGlobal
   public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            //调整布局参数，并设置token
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        }
        ViewRootImpl root;
        View panelParentView = null;
        synchronized (mLock) {
            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    //如果待删除的view中有当前view，删除它
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                }
                // The previous removeView() had not completed executing. Now it has.
               //之前移除View并没有完成删除操作，现在正式删除该view
            }

            //如果这是一个子窗口个(popupWindow)，找到它的父窗口。
            //最本质的作用是使用父窗口的token(viewRootImpl的W类，也就是IWindow)
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                    //在源码中token一般代表的是Binder对象，作用于IPC进程间数据通讯。并且它也包含着此次通讯所需要的信息，
                    //在ViewRootImpl里，token用来表示mWindow(W类，即IWindow)，并且在WmS中只有符合要求的token才能让
                    //Window正常显示.
                        panelParentView = mViews.get(i);
                    }
                }
            }
            //创建ViewRootImpl，并且将view与之绑定
            root = new ViewRootImpl(view.getContext(), display);
            view.setLayoutParams(wparams);
            mViews.add(view);//将当前view添加到mViews集合中，mViews存储所有Window对应的View
            mRoots.add(root);//将当前ViewRootImpl添加到mRoots集合中，mRoots存储所有Window对应的ViewRootImpl
            mParams.add(wparams);//将当前window的params添加到mParams集合中，存储所有Window对应的布局参数
        }
          ......
            //通过ViewRootImpl的setView方法，完成view的绘制流程，并添加到window上。
            root.setView(view, wparams, panelParentView);
    }
```

最最重要的是：root.setView(view, wparams, panelParentView); 一方面触发绘制流程，一方面把view添加到window上。


### 4.3.4 ViewRootImpl
https://www.jianshu.com/p/9da7bfe18374
ViewRootImpl的功能可不只是绘制，它还有事件分发的功能
https://www.jianshu.com/p/9e6c54739217

下面看看ViewRootImpl的setView：

```java
//ViewRootImpl
  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
                int res;
                 //在 Window add之前调用，确保 UI 布局绘制完成 --> measure , layout , draw
                requestLayout();//View的绘制流程
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    //创建InputChannel
                    mInputChannel = new InputChannel();
                }
                try {
                    //通过WindowSession进行IPC调用，将View添加到Window上
                    //mWindow即W类，用来接收WmS信息
                    //同时通过InputChannel接收触摸事件回调
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                }
                .....
                    //处理触摸事件回调
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                .....
    }
```

在ViewRootImpl的setView()方法里，
1.执行requestLayout()方法完成view的绘制流程（之后会讲）  
2.通过WindowSession将View和InputChannel添加到WmS中，从而将View添加到Window上并且接收触摸事件。这是一次IPC 过程。  
那么接下来看看这个IPC过程

```java
//ViewRootImpl的setView方法中：
mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
        getHostVisibility(), mDisplay.getDisplayId(),
        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
        mAttachInfo.mOutsets, mInputChannel);
```

mWindowSession：类型是interface IWindowSession

```java
//WindowManagerGlobal
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                IWindowManager windowManager = getWindowManagerService();
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        },
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}
```

我们看到了getWindowManagerService(); 获取了WMS , 那么再看下windowManager.openSession返回值就是sWindowSession

```java
@Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```

IWindowSession的真正实现类是Session，他是一个Binder. 那么Session的addToDisplay:

```java
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

从这知道了，最终是WMS执行addWindow操作.




WMS执行addWindow部分代码有点多，本篇就不铺开说了，不然篇幅就太长了，之后再说，看下如下的流程图：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081545047.png)

总结起来：WMS中 addWindow流程就几点：  
1.通过type和 token对window进行分类和验证，确保其有效性。  
2.构造WindowState与Window一一对应。  
3.通过token对window进行分组。  
4.对window分配层级。

那么到这里，window添加view的过程就结束了。

五、总结  
下面用一张图来总结下Activity、PhoneWindow、 DecorView 、WindowManagerGlobal 、ViewRootImpl 、Wms 以及WindowState之间的关系：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081546677.png)

Activity在attach的时候，创建了一个PhoneWindow对象，并且实现了Window的Callback接口，这样activity就和window绑定在了一起，通过setContentView，创建DecorView，并解析好视图树加载到DecorView的contentView部分，WindowManagerGlobal一个进程只有唯一一个，对当前进程内所有的视图进行统一管理，其中包括ViewRootImpl，它主要做两件事情，先触发view绘制流程，再通过IPC 把view添加到window上。

另外这是添加视图的方法执行时序图：

### 4.3.5 时序图
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303110936914.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081546950.png)



### 4.3.6 类图
```plantuml
class WindowManagerService extends IWindowManager.Stub implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
        }

interface ViewManager{
 void addView(View view, ViewGroup.LayoutParams params);  
 void updateViewLayout(View view, ViewGroup.LayoutParams params);  
 void removeView(View view);
}
interface WindowManager extends ViewManager {

}
class WindowManagerImpl implements WindowManager {
	WindowManagerGlobal mGlobal;
}	
WindowManagerImpl -> WindowManagerGlobal

class WindowManagerGlobal {
	ViewRootImpl mRoots;
}
WindowManagerGlobal -> ViewRootImpl
```


```plantuml
interface IWindowSession extends android.os.IInterface{
	int addToDisplay();
}
interface IWindowManager extends android.os.IInterface{
   IWindowSession openSession();
}
class WindowManagerService extends IWindowManager.Stub implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
        }
```

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303110936685.png)


至于Window的删除和更新过程，举一反三，也是使用WindowManagerGlobal对ViewRootImpl的操作，最终也是通过Session的IPC跨进程通信通知到WmS。整个过程的本质都是同出一辙的。下一节接着讲DecorView布局的加载流程。

注意：IWindow 1.aidl、IWindowManager.aidl 等都是 aild 文件
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081706792.png)



## 4.4 参考  
[https://blog.csdn.net/qian520ao/article/details/78555397?locationNum=7&fps=1](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fqian520ao%2Farticle%2Fdetails%2F78555397%3FlocationNum%3D7%26fps%3D1)  
[https://www.jianshu.com/p/effaff9ab9f2](https://www.jianshu.com/p/effaff9ab9f2)  
[https://blog.csdn.net/freekiteyu/article/details/79408969](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Ffreekiteyu%2Farticle%2Fdetails%2F79408969)


# 5 DecorView布局加载流程
上篇我们了解了window的创建过程和添加视图的流程，但是顶级视图DecorView是怎么被加载的呢？其实这个过程非常简单，分析下setContentView的过程，一切就明了了。

## 5.1 关系介绍
```java
public class PhoneWindow extends Window implements MenuBuilder.Callback { 
      ... 　　
      //窗口顶层
    View private DecorView mDecor; 　　
    //所有自定义View的根View, id="@android:id/content" 
    private ViewGroup mContentParent; 　　
     ... 
} 
```

先交代下PhoneWindow 与DecorView 以及mContentParent的关系：mDecor是窗口顶层视图，mContentParent是mDecor上content framelayout的父容器，用来装xml解析出来的view树。

## 5.2 setContentView流程
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081550913.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303110949317.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303110951802.png)

### 5.2.1 setContentView

```java
// Activity.java 
public void setContentView(@LayoutRes int layoutResID) { 
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
 }
```

然而setContentView是window的一个抽象方法，真正实现类是PhoneWindow. 这个方法有3个重载：

```java
@Override
public void setContentView(int layoutResID) {
   ...
        mLayoutInflater.inflate(layoutResID, mContentParent);
   ...
}

@Override
public void setContentView(View view) {
    setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
}

@Override
public void setContentView(View view, ViewGroup.LayoutParams params) {
    ...
        mContentParent.addView(view, params);
    ...
}
```

看上去是3个，其实是2个，这两个重载的区别的，一个是解析xml视图，一个是直接传入视图。那么我们就看相对复杂点的xml解析的。

```java
//PhoneWindow


    @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {

//1.初始化
        //创建DecorView对象和mContentParent对象 ,并将mContentParent关联到DecorView上
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();//Activity转场动画相关
        }

//2.填充Layout
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);//Activity转场动画相关
        } else {
        //将Activity设置的布局文件，加载到mContentParent中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }

        //让DecorView的内容区域延伸到systemUi下方，防止在扩展时被覆盖，达到全屏、沉浸等不同体验效果。
        mContentParent.requestApplyInsets();

//3. 通知Activity布局改变
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {

        //触发Activity的onContentChanged方法  
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

(1) 初始化 : Activity第一次调用setContentView，则会调用installDecor()方法**创建DecorView对象**和mContentParent对象。FEATURE_CONTENT_TRANSITIONS表示是否使用转场动画。如果内容已经加载过，并且不需要动画，则会调用removeAllViews移除内容以便重新填充Layout。  
(2) **填充Layout** : 初始化完毕，如果设置了FEATURE_CONTENT_TRANSITIONS，就会创建Scene完成转场动画。否则使用布局填充器将布局文件填充至mContentParent。到此为止，Activity的布局文件已经添加到DecorView里面了，所以可以理解Activity的setContentView方法的由来，因为布局文件是添加到DecorView的mContentParent中，所以方法名为setContentView无可厚非。

那么核心方法就两个：installDecor() 和 mLayoutInflater.inflate(layoutResID, mContentParent)，下面来一一介绍。

### 5.2.2 installDecor

```java
//PhoneWindow
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        mDecor = generateDecor(-1);// 创建 DecorView
       mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
         //根据窗口的风格修饰，选择对应的修饰布局文件，并且将id为content的FrameLayout赋值给mContentParent 
        mContentParent = generateLayout(mDecor);
       …  //初始化一堆属性值
    }
}
```

generateDecor(-1）很简单，就是new DecorView  
重点关注下 mContentParent = generateLayout(mDecor);

```cpp
protected ViewGroup generateLayout(DecorView decor) {
   //1,为Activity配置相应属性，即android：theme=“”，PhoneWindow对象调用getWindowStyle()方法获取值。
   TypedArray a = getWindowStyle();
    ...
   //2,获取窗口Features, 设置相应的修饰布局文件，这些xml文件位于frameworks/base/core/res/res/layout下
   int layoutResource;
   int features = getLocalFeatures();//指定requestFeature()指定窗口修饰符，PhoneWindow对象调用getLocalFeature()方法获取值；
    ...
    mDecor.startChanging();
    //3, 将上面选定的布局文件inflate为View树，添加到decorView中
   mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    //4,将窗口修饰布局文件中id="@android:id/content"的View赋值给mContentParent, 后续自定义的view/layout都将是其子View
   ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    ...
    mDecor.finishChanging();
   return contentParent;
}
```

installDecor() 做了这么几件事：  
1） **创建一个DecorView的对象**mDecor，该mDecor对象将作为整个应用窗口的根视图。  
2） 配置不同窗口修饰属性（style theme等）。  
3） **将DecorView布局中id为content**的FrameLayou的Viewt赋值给mContentParent  
至此，DecorView 的 contentView 大容器已经设置完成， 但是里面并没有内容，原因是用户自定义的xml文件还没有解析加载到contentView上。

### 5.2.3 LayoutInflater.inflate(layoutResID, mContentParent)解析加载视图
获取LayoutInflater实例两种方式：

```java
LayoutInflater lif = LayoutInflater.from(Context context);  

LayoutInflater lif = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE); 
```

from只不过是对下面方式的一种封装而已。来看看inflate方法，然后你会发现无论哪个inflate的重载方法最后都调运了inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot)方法，那么分析下这个方法就好了：

```java
//LayoutInflater
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
       //定义返回值，初始化为传入的形参root
        View result = root;
        try {
            //寻找根结点，开始xml pull解析过程
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }
            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }
            final String name = parser.getName();
            if (DEBUG) {
                System.out.println("**************************");
                System.out.println("Creating root view: "
                        + name);
                System.out.println("**************************");
            }
            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                ViewGroup.LayoutParams params = null;
                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }
                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }
                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);
                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }
                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
        } catch (XmlPullParserException e) {
            final InflateException ie = new InflateException(e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (Exception e) {
            final InflateException ie = new InflateException(parser.getPositionDescription()
                    + ": " + e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        return result;
    }
}
```

典型的pull解析的方式，深度优先地递归解析xml，一层层添加到root view上，最终返回root view.解析的部分大致包含两点：
1.解析出View对象，
2.解析View对应的Params，并设置给View。

而我们看到LayoutInflater.inflate(layoutResID, mContentParent)，传进去的是mContentParent，也就是最终root view就是mContentParent。xml布局解析完毕，且add到了mContentParent上。

至此DecorView视图组建完成。


### 5.2.4 DecorView的添加

当启动Activity调运完ActivityThread的main方法之后，接着调用ActivityThread类performLaunchActivity来创建要启动的Activity组件，在创建Activity组件的过程中，还会为该Activity组件创建窗口对象和视图对象；接着Activity组件创建完成之后，通过调用ActivityThread类的**handleResumeActivity**将它激活。

我们从ActivityThread的handleLaunchActivity()方法开始

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
...
Activity a = performLaunchActivity(r, customIntent);
...
}
``
performLaunchActivity中：

通过Instrumentation.newActivity的方法创建Activity, 在之后activity执行attach方法会初始化PhoneWindow.

另外在
``
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
…
if (r.window == null && !a.mFinished && willBeVisible) {
    r.window = r.activity.getWindow();
    View decor = r.window.getDecorView();
    decor.setVisibility(View.INVISIBLE);
    ViewManager wm = a.getWindowManager();
    WindowManager.LayoutParams l = r.window.getAttributes();
   ...
    if (a.mVisibleFromClient && !a.mWindowAdded) {
        a.mWindowAdded = true;
        wm.addView(decor, l);
    }
}
…
     if (r.activity.mVisibleFromClient) {
         r.activity.makeVisible();
     }
...
}
```

再看下Activity的makeVisible方法：

```java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE); //显示DecorView
}
```

首先我们都看到了

```java
  ViewManager wm = a.getWindowManager();
   wm.addView(decor, l);
```

添加了decorView， 追下下实现类：WindowManagerImpl ，看看addView方法：

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

这个mGlobal 即：WindowManagerGlobal, 那就看他的addView方法：

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
   ...
    ViewRootImpl root;
    View panelParentView = null;
   ...
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }
    // do this last because it fires off messages to start doing things
    try {
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
       ...
    }
}
```

创建ViewRootImpl, 并将DecorView添加到ViewRootImpl上，那么添加的动作是setView 看下：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) { 
    synchronized (this) { 
         if (mView == null) { 
             mView = view; 
            //发起绘制流程 
            requestLayout(); 
             … 
            //设置ViewRootImpl为DecorView的父控件 
           view.assignParent(this); 
            ...
        } 
     }
 }
```

好了到这大概已经清楚了整个布局加载流程，其实非常简单，就是创建DecorView，并把xml的View树解析出来，加到DecorView上，形成完整的View组件的过程。下一节接着讲从ViewRootImpl开始的绘制流程。

## 5.3 参考
[https://blog.csdn.net/yanbober/article/details/45970721](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fyanbober%2Farticle%2Fdetails%2F45970721)  
[https://blog.csdn.net/zhangcanyan/article/details/52973127](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fzhangcanyan%2Farticle%2Fdetails%2F52973127)  
[https://www.jianshu.com/p/16e978f9d421](https://www.jianshu.com/p/16e978f9d421)

# 6 屏幕刷新机制
[[4-屏幕刷新机制]]

# 7 View绘制流程
接上篇 [绘制优化-原理篇2-DecorView布局加载流程](https://www.jianshu.com/p/b3a1ea7923e7) 讲到的ViewRootImpl，在ViewRootImpl的setView()方法里主要做两件事：  
1.执行requestLayout()方法完成view的绘制流程  
2.通过WindowSession将View和InputChannel添加到WmS中，从而将View添加到Window上并且接收触摸事件。

2的部分 window加载视图已经介绍了，那么今天就来讲讲1的部分：执行requestLayout()方法完成view的绘制流程

```cpp
//ViewRootImpl
  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
                 //在 Window add之前调用，确保 UI 布局绘制完成 --> measure , layout , draw
                requestLayout();//View的绘制流程
                ...
                    //通过WindowSession进行IPC调用，将View添加到Window上
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                }
                ...
    }
```

## 7.1 从requestLayout开始
从requestLayout代码一层层往下追（具体源码不贴了，非常简单），最终确认view的绘制流程是从performTraversals开始。顺一下整个流程：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081552184.png)

### 7.1.1 performTraversals

```java
private void performTraversals() {
     //获得view宽高的测量规格，mWidth和mHeight表示窗口的宽高，lp.width和lp.height表示DecorView根布局宽和高
     WindowManager.LayoutParams lp = mWindowAttributes;
     ...
        //顶层视图DecorView所需要窗口的宽度和高度
        int desiredWindowWidth;
        int desiredWindowHeight;
     ...
        //在构造方法中mFirst已经设置为true，表示是否是第一次绘制DecorView
        if (mFirst) {
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            //如果窗口的类型是有状态栏的，那么顶层视图DecorView所需要窗口的宽度和高度就是除了状态栏
            if (lp.type == WindowManager.LayoutParams.TYPE_STATUS_BAR_PANEL
                    || lp.type == WindowManager.LayoutParams.TYPE_INPUT_METHOD) {
                // NOTE -- system code, won't try to do compat mode.
                Point size = new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth = size.x;
                desiredWindowHeight = size.y;
            } else {//否则顶层视图DecorView所需要窗口的宽度和高度就是整个屏幕的宽高
                DisplayMetrics packageMetrics =
                    mView.getContext().getResources().getDisplayMetrics();
                desiredWindowWidth = packageMetrics.widthPixels;
                desiredWindowHeight = packageMetrics.heightPixels;
            }
    }
    ...
     int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
     int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
     ...
     // Ask host how big it wants to be
     //执行测量操作
     performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
      ...
     //执行布局操作
    performLayout(lp, mWidth, mHeight);
     ...
     //执行绘制操作
     performDraw();
}
```

performTraversals()中做了非常多的处理，代码接近800行，这里我们重点关注绘制相关流程。

### 7.1.2 MeasureSpec
在分析绘制过程之前，我们需要先了解MeasureSpec，它是干什么的呢？简而言之，MeasureSpec 是View的尺寸一种封装手段。

MeasureSpec代表一个32位int值，高2位代表SpecMode,低30位代表SepcSize. 这样的打包方式好处是避免过多的对象内存分配。为了方便操作，其提供了打包和解包的方法:

```java
public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
        public static final int EXACTLY     = 1 << MODE_SHIFT;
        public static final int AT_MOST     = 2 << MODE_SHIFT;
...
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                  @MeasureSpecMode int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
...
public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK); //高2位运算
        }
public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);//低30位运算
        }
}
```

getMode方法中ModeMask 为 0x3 << 30 转换成二进制为 0011 << 30 ，也就是向左移动30位 则 ModeMask高两位为1，低三十位为0，整形measureSpec 为32位， measureSpec & mode_mask 就是高2位的运算，getSize方法中 ~Mode_MASK 则是除了高两位为0 外剩下的低30位均为 1，那么和measure 进行 & 运算就是在求得低30为中存储的值。

**SpecMode：测量模式**
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081552382.png)


**SpecSize：对应某种测量模式下的尺寸大小**

下面针对DecorView和普通View分别来看看其MeasureSpec的组成：

```cpp
//ViewRootImpl
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
 DecorView, 其MeasureSpec由窗口尺寸和其自身LayoutParams共同决定，
}

//ViewGroup
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
   普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams共同决定。
}
```

对普通View的MeasureSpec的创建规则进行总结：

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081553883.png)

这个表怎么用呢？举个例子：  
如果View在布局中使用wrap_content,那么它的specMode是AT_MOST,这种模式下，它的宽高为specSize, 而查表可得View的specSize是parentSize，而parentSize是当前父容器剩余空间大小，这种效果和在布局中使用match_parent完全一致，所以如果是对尺寸有具体要求的自定义控件需要指定specSize大小。

注：  
LayoutParams类是用于子视图向父视图传达自己尺寸意愿的一个参数包，包含了Layout的高、宽信息。LayoutParams在LayoutInflater.inflater过程中与View一起被解析成对象，保存在WindowManagerGlobal集合中。


## 7.2 View绘制流程

performTraversals里面执行了三个方法，分别是performMeasure()、performLayout()、performDraw()这三个方法，这三个方法分别完成DecorView的measure、layout、和draw这三大流程，其中performMeasure()中会调用measure()方法，在measure()方法中又会调用onMeasure()方法，在onMeasure()方法中会对所有子元素进行measure过程，这个时候measure流程就从父容器传递到子元素中了，这样就完成了一次measure过程。接着子元素会重复父容器的measure过程，如此反复就实现了从DecorView开始对整个View树的遍历测量，measure过程就这样完成了。同理，performLayout()和performDraw()也是类似的传递流程。针对performTraveals()的大致流程，可以用以下流程图来表示：

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081553969.png)

以上的流程图只是一个为了便于理解而简化版的流程，真正的流程应该分为以下五个工作阶段：

-   **预测量阶段**：这是进入performTraversals()方法后的第一个阶段，它会对View树进行第一次测量。在此阶段中将会计算出View树为显示其内容所需的尺寸，即期望的窗口尺寸。（调用measureHierarchy()）
   
-   **窗口布局阶段**：根据预测量的结果，通过IWindowSession.relayout()方法向WMS请求调整窗口的尺寸等属性，这将引发WMS对窗口进行重新布局，并将布局结果返回给ViewRootImpl。（调用relayoutWindow()）
   
-   **测量阶段**：预测量的结果是View树所期望的窗口尺寸。然而由于在WMS中影响窗口布局的因素很多，WMS不一定会将窗口准确地布局为View树所要求的尺寸，而迫于WMS作为系统服务的强势地位，View树不得不接受WMS的布局结果。因此在这一阶段，performTraversals()将以窗口的实际尺寸对View树进行最终测量。（调用performMeasure()）
   
-   **布局阶段**：完成最终测量之后便可以对View树进行布局了。（调用performLayout()）
    
-   **绘制阶段**：这是performTraversals()的最终阶段。确定了控件的位置与尺寸后，便可以对View树进行绘制了。（调用performDraw()）   

下面分别来阐述：

### 7.2.1 预测量阶段（performTraversals（））
这个阶段在performTraversals中最先发生，对View树进行第一次测量，会判断当前期望窗口尺寸是否能满足布局要求。

```java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        int childWidthMeasureSpec;
        int childHeightMeasureSpec;
        // 表示测量结果是否可能导致窗口的尺寸发生变化
        boolean windowSizeMayChange = false;
        //goodMeasure表示了测量是否能满足View树充分显示内容的要求
        boolean goodMeasure = false;
        //测量协商仅发生在LayoutParams.width被指定为WRAP_CONTENT的情况下
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
            //第一次协商。measureHierarchy()使用它最期望的宽度限制进行测量。
            //这一宽度限制定义为一个系统资源。
            //可以在frameworks/base/core/res/res/values/config.xml找到它的定义
            final DisplayMetrics packageMetrics = res.getDisplayMetrics();
            res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
            // 宽度限制被存放在baseSize中
            int baseSize = 0;
            if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
                baseSize = (int)mTmpValue.getDimension(packageMetrics);
            }
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
                //第一次测量。调用performMeasure()进行测量
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                //View树的测量结果可以通过mView的getmeasuredWidthAndState()方法获取。
                //View树对这个测量结果不满意，则会在返回值中添加MEASURED_STATE_TOO_SMALL位
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    goodMeasure = true;  // 控件树对测量结果满意，测量完成
                } else {
                  //第二次协商。上次的测量结果表明View树认为measureHierarchy()给予的宽度太小，在此
                  //在此适当地放宽对宽度的限制，使用最大宽度与期望宽度的中间值作为宽度限制
                    baseSize = (baseSize+desiredWindowWidth)/2;
                    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                  //第二次测量
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                  // 再次检查控件树是否满足此次测量
                    if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                     // 控件树对测量结果满意，测量完成
                        goodMeasure = true;
                    }
                }
            }
        }
        if (!goodMeasure) {
          //最终测量。当View树对上述两次协商的结果都不满意时，measureHierarchy()放弃所有限制
          //做最终测量。这一次将不再检查控件树是否满意了，因为即便其不满意，measurehierarchy()也没
          //有更多的空间供其使用了
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
          //如果测量结果与ViewRootImpl中当前的窗口尺寸不一致，则表明随后可能有必要进行窗口尺寸的调整
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }
        // 返回窗口尺寸是否可能需要发生变化
        return windowSizeMayChange;
    }
```

measureHierarchy()方法最终也是调用了performMeasure()方法对View树进行测量，只是多了协商测量的过程。

### 7.2.2 窗口布局阶段（relayoutWindow（））
调用relayoutWindow()来请求WindowManagerService服务计算Activity窗口的大小以及过扫描区域边衬大小和可见区域边衬大小。计算完毕之后，Activity窗口的大小就会保存在成员变量mWinFrame中，而Activity窗口的内容区域边衬大小和可见区域边衬大小分别保存在ViewRoot类的成员变量mPendingOverscanInsets和mPendingVisibleInsets中。

这部分不在这细讲了，有耐心的可以把老罗的文章看完：[Android窗口管理服务WindowManagerService计算Activity窗口大小的过程分析](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2FLuoshengyang%2Farticle%2Fdetails%2F8479101)

### 7.2.3 测量过程(performMeasure（）)
measure整体执行流程：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081555806.png)

WMS的布局结果已经确定了，不管是否满意都得开始终极布局过程了，下面介绍下measure:  
measure是对View进程测量，确定各View的尺寸的过程,这个过程分View和ViewGroup两种情况来看，对于View，通过measure完成自身的测量就行了，而ViewGroup除了完成自身的测量外，还需要遍历去调用所有子view的measure方法，各个子view递归去执行这个过程。

那么先从performMeasure开始：
```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

这里的mView是 ViewRootImpl setView传进来的rootView.  
接下来我们看下View的measure()方法
```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
     ...
    //根据widthMeasureSpec和heightMeasureSpec计算key值，在下面用key值作为键，缓存我们测量得到的结果
    long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
    if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
    …
    //forceLayout 是通过上次的mPrivateFlags标记位来判断这次是否需要触发重绘
    //View中有个forceLayout()方法可以设置mPrivateFlags.
   // needsLayout 简单看就是spec发生了某些规则约束下的变化
    if (forceLayout || needsLayout) {
        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
        resolveRtlPropertiesIfNeeded();
        //在View真正进行测量之前，View还想进一步确认能不能从已有的缓存mMeasureCache中读取缓存过的测量结果 //如果是强制layout导致的测量，那么将cacheIndex设置为-1，即不从缓存中读取测量结果 //如果不是强制layout导致的测量，那么我们就用上面根据measureSpec计算出来的key值作为缓存索引cacheIndex,这时候有可能找到相应的值,找到就返回对应索引;也可能找不到,找不到就返回-1
        int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            //在缓存中找不到相应的值或者需要忽略缓存结果的时候,重新测量一次 //此处调用onMeasure方法，并把尺寸限 制条件widthMeasureSpec和heightMeasureSpec传入进去 //onMeasure方法中将会进行实际的测量工作，并把测量的结果保存到成员变量中
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            //如果运行到此处，那么表示当前的条件允许View从缓存成员变量mMeasureCache中读取测量过的结果
           //用上面得到的cacheIndex从缓存mMeasureCache中取出值，不必在调用onMeasure方法进行测量了
            long value = mMeasureCache.valueAt(cacheIndex);
          //一旦我们从缓存中读到值，我们就可以调用setMeasuredDimensionRaw方法将当前测量的结果保存到成员变量中        
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
          ...
        }
      ...
    }
    ...
}
```

先判断一下是否有必要进行测量操作,如果有，先看是否能在缓存mMeasureCache中找到上次的测量结果,如果找到了那直接从缓存中获取就可以了，如果找不到，那么乖乖地调用onMeasure()方法去完成实际的测量工作，并且将尺寸限制条件widthMeasureSpec和heightMeasureSpec传递给onMeasure()方法。

另外，measure()这个方法是final的，因此我们无法在子类中去重写这个方法，说明Android是不允许我们改变View的measure框架. 主要看onMeasure()方法，这里才是真正去测量并设置View大小的地方。

#### 7.2.3.1 View的measure过程：
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
   setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

setMeasuredDimension方法会设置View宽高的测量值，因此主要看下getDefaultSize方法：

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

很显然看出：  
AT_MOST 和 EXACTLY两种情况返回的就是specSize。

UNSPECIFIED返回的是size, 即getSuggestedMinimumWidth 和 getSuggestedMinimumHeight的返回值。

```csharp
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

mMinWidth 对应于android:minWidth属性指定的值。总结：如果View没有设置背景，那么返回mMinWidth的值，否则返回mMinWidth和背景最小宽度的最大值。

#### 7.2.3.2 ViewgGroup的measure过程：
ViewGroup 继承自 View，我们知道View的 measure是 final方法，那这个方法是肯定会走的，但是具体实现是在onMeasure中，ViewGroup提供了几个方法来帮助ViewGroup的子类来实现onMeasure逻辑，包括：

```java
protected void measureChild(View child, int parentWidthMeasureSpec,
       int parentHeightMeasureSpec) {
   final LayoutParams lp = child.getLayoutParams();
   final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
           mPaddingLeft + mPaddingRight, lp.width);
   final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
           mPaddingTop + mPaddingBottom, lp.height);
   child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

protected void measureChildWithMargins(View child,
       int parentWidthMeasureSpec, int widthUsed,
       int parentHeightMeasureSpec, int heightUsed) {
   final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
   final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
           mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                   + widthUsed, lp.width);
   final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
           mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                   + heightUsed, lp.height);
   child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

仔细看其实最终还是让child去执行自己对于的measure，只是getChildMeasureSpec有差别，这里加上了margin 和 padding.

具体onMeasure的实现可以参考LinearLayout、FrameLayout、RelativeLayout等。

另外需要关注的是ViewGroup 的 getChildMeasureSpec方法，我们从上面代码中很明显看出，传入的Spec是父容器的measureSpec

```cpp
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
   int specMode = MeasureSpec.getMode(spec);
   int specSize = MeasureSpec.getSize(spec);
   int size = Math.max(0, specSize - padding);
   int resultSize = 0;
   int resultMode = 0;
   switch (specMode) {
   // Parent has imposed an exact size on us
   case MeasureSpec.EXACTLY:
       if (childDimension >= 0) {
           resultSize = childDimension;
           resultMode = MeasureSpec.EXACTLY;
       } else if (childDimension == LayoutParams.MATCH_PARENT) {
           // Child wants to be our size. So be it.
           resultSize = size;
           resultMode = MeasureSpec.EXACTLY;
       } else if (childDimension == LayoutParams.WRAP_CONTENT) {
           // Child wants to determine its own size. It can't be
           // bigger than us.
           resultSize = size;
           resultMode = MeasureSpec.AT_MOST;
       }
       break;
   // Parent has imposed a maximum size on us
   case MeasureSpec.AT_MOST:
       if (childDimension >= 0) {
           // Child wants a specific size... so be it
           resultSize = childDimension;
           resultMode = MeasureSpec.EXACTLY;
       } else if (childDimension == LayoutParams.MATCH_PARENT) {
           // Child wants to be our size, but our size is not fixed.
           // Constrain child to not be bigger than us.
           resultSize = size;
           resultMode = MeasureSpec.AT_MOST;
       } else if (childDimension == LayoutParams.WRAP_CONTENT) {
           // Child wants to determine its own size. It can't be
           // bigger than us.
           resultSize = size;
           resultMode = MeasureSpec.AT_MOST;
       }
       break;
   // Parent asked to see how big we want to be
   case MeasureSpec.UNSPECIFIED:
       if (childDimension >= 0) {
           // Child wants a specific size... let him have it
           resultSize = childDimension;
           resultMode = MeasureSpec.EXACTLY;
       } else if (childDimension == LayoutParams.MATCH_PARENT) {
           // Child wants to be our size... find out how big it should
           // be
           resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
           resultMode = MeasureSpec.UNSPECIFIED;
       } else if (childDimension == LayoutParams.WRAP_CONTENT) {
           // Child wants to determine its own size.... find out how
           // big it should be
           resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
           resultMode = MeasureSpec.UNSPECIFIED;
       }
       break;
   }
   //noinspection ResourceType
   return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

很明显看出来，对于普通View来说，getChildMeasureSpec的逻辑是通过其父View提供的MeasureSpec参数得到specMode和specSize，然后根据计算出来的specMode以及子View的childDimension（layout_width或layout_height）来计算自身的measureSpec 。

**measure总结：**

-   MeasureSpec 由specMode和specSize组成：  
    DecorView, 其MeasureSpec由窗口尺寸和其自身LayoutParams共同决定。  
    普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams共同决定。
    
-   View的measure方法是final的，不允许重载，View子类只能重载onMeasure来完成自己的测量逻辑。
    
-   ViewGroup类提供了measureChild，measureChild和measureChildWithMargins方法，以及getChildMeasureSpec方法 ，供具体实现ViewGroup的子类重写onMeasure的时候方便使用。
    
-   只要是ViewGroup的子类就必须要求LayoutParams继承子MarginLayoutParams，否则无法使用layout_margin参数。
    
-   使用View的getMeasuredWidth()和getMeasuredHeight()方法来获取View测量的宽高，必须保证这两个方法在onMeasure流程之后被调用才能返回有效值。  
    比较常用的方式：  
    view.post(runnable)  
    view.measure(0,0)之后 get
    

### 7.2.4 布局过程 (performLayout（）)
layout的整体执行流程：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081556140.png)


Layout的作用是ViewGroup用来确定子view的位置，当ViewGroup的位置被确定之后，它在onLayout中会遍历所有子view并调用其layout方法，在layout方法中onLayout又被调用。

先从performLayout看起：
```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
           int desiredWindowHeight) {
       ...
       //标记当前开始布局
       mInLayout = true;
       //mView就是DecorView
       final View host = mView;
       ...
       //DecorView请求布局 layout参数分别是 左 上 右 下
       host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
       //标记布局结束
       mInLayout = false;
       ...
}
```

跟踪代码进入View类的layout方法
```csharp
public void layout(int l, int t, int r, int b) {
       //判断是否需要重新测量
       if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
           onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
           mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
       }
       //保存上一次View的四个位置
       int oldL = mLeft;
       int oldT = mTop;
       int oldB = mBottom;
       int oldR = mRight;
       //设置当前视图View的左，顶，右，底的位置，并且判断布局是否有改变
       boolean changed = isLayoutModeOptical(mParent) ?
               setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
       //如果布局有改变，条件成立，则视图View重新布局
           if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
           //调用onLayout，将具体布局逻辑留给子类实现
           onLayout(changed, l, t, r, b);
           mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
           ListenerInfo li = mListenerInfo;
           if (li != null && li.mOnLayoutChangeListeners != null) {
               ArrayList<OnLayoutChangeListener> listenersCopy =
                       (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
               int numListeners = listenersCopy.size();
               for (int i = 0; i < numListeners; ++i) {
                   listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
               }
           }
       }
       mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
       mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
   }
```

首先关注下需要重新layout的条件：
```java
boolean changed = isLayoutModeOptical(mParent) ?
               setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
```

其中setOpticalFrame内部也会调用setFrame，所以就看下setFrame好了：
```java
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;
    ...
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
        // Remember our drawn bit
        int drawn = mPrivateFlags & PFLAG_DRAWN;
        int oldWidth = mRight - mLeft;
        int oldHeight = mBottom - mTop;
        int newWidth = right - left;
        int newHeight = bottom - top;
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);
        // Invalidate our old position
        invalidate(sizeChanged);
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
       ...
    }
    return changed;
}
```

通过setFrame方法设定View的四个顶点的位置，并更新本地值，同时判断顶点位置较之前是否有变化，并return是否有变化的boolean值，如果有变化还会执行invalidate(sizeChanged)。

然后，咱们再看看onLayout方法：  
View中的onLayout是个空方法，可实现可不实现
```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```

ViewGroup中是个抽象方法，子类必须实现

```java
@Override
protected abstract void onLayout(boolean changed,int l, int t, int r, int b);
```

下面以RelativeLayout为例，对onLayout具体实现做简单的分析：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
   //  The layout has actually already been performed and the positions
   //  cached.  Apply the cached values to the children.
   final int count = getChildCount();
   for (int i = 0; i < count; i++) {
       View child = getChildAt(i);
       if (child.getVisibility() != GONE) {//只有不为GONE的才会执行布局
           RelativeLayout.LayoutParams st =
                   (RelativeLayout.LayoutParams) child.getLayoutParams();
           child.layout(st.mLeft, st.mTop, st.mRight, st.mBottom);
       }
   }
}
```

遍历所有子view，并通过其LayoutParams 获取四个方向的位置值，将位置信息传入子view的layout方法进行布局。

**layout总结:**
-   Layout的作用是ViewGroup用来确定子view的位置, 是ViewGroup需要干的活，View不需要，所以View中是空方法，而ViewGroup中是抽象方法，但是View你也可以重写，大多数是利用这个生命周期阶段加写逻辑操作。
   
-   当我们的视图View在布局中使用 android:visibility=”gone” 属性时，是不占据屏幕空间的，因为在布局时ViewGroup会遍历每个子视图View，判断当前子视图View是否设置了 Visibility==GONE,如果设置了，当前子视图View就不会添加到父容器上，因此也就不占据屏幕空间。具体可以参考RelativeLayout的onLayout.
   
-   必须在View布局完之后调用getHeight( )和getWidth( )方法获取到的View的宽高才大于0.
   

### 7.2.5 绘制过程 (performDraw（）)
draw整体执行流程：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081556187.png)


Draw作用是将View绘制到屏幕上.过程相对比较简单。  
draw是从performDraw开始
```cpp
//ViewRootImpl
private void performDraw() {
       ...
       draw(fullRedrawNeeded);
       ...
}
```

然后看ViewRootImpl的draw方法：

```java
 //ViewRootImpl
   private void draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
     ...
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                    return;
                }
     ...
    }
```

再看ViewRootImpl的drawSoftware方法：

```java
 //ViewRootImpl
  private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {
...
 mView.draw(canvas);
...
}
```

最终在drawSoftware方法中，会走到View的draw并传入了canvas画布。这部分先不细说，之后的Surface部分会分析。

那么接着往下的话就是真正View绘制的部分了:

```swift
   //View
   public void draw(Canvas canvas) {
       ......
       /*
        * Draw traversal performs several drawing steps which must be executed
        * in the appropriate order:
        *
        *      1\. Draw the background
        *      2\. If necessary, save the canvas' layers to prepare for fading
        *      3\. Draw view's content
        *      4\. Draw children
        *      5\. If necessary, draw the fading edges and restore layers
        *      6\. Draw decorations (scrollbars for instance)
        */
       // Step 1, draw the background, if needed
       ......
       if (!dirtyOpaque) {
           drawBackground(canvas);
       }
       // skip step 2 & 5 if possible (common case)
       ......
       // Step 2, save the canvas' layers
       ......
           if (drawTop) {
               canvas.saveLayer(left, top, right, top + length, null, flags);
           }
       ......
       // Step 3, draw the content
       if (!dirtyOpaque) onDraw(canvas);
       // Step 4, draw the children
       dispatchDraw(canvas);
       // Step 5, draw the fade effect and restore layers
       ......
       if (drawTop) {
           matrix.setScale(1, fadeHeight * topFadeStrength);
           matrix.postTranslate(left, top);
           fade.setLocalMatrix(matrix);
           p.setShader(fade);
           canvas.drawRect(left, top, right, top + length, p);
       }
       ......
       // Step 6, draw decorations (scrollbars)
       onDrawScrollBars(canvas);
       ......
   }
```

从摘要可以看出，绘制过程分如下几步：
-   **绘制背景 background.draw(canvas)**

```java
   private void drawBackground(Canvas canvas) {
       //获取xml中通过android:background属性或者代码中setBackgroundColor()、setBackgroundResource()等方法进行赋值的背景Drawable
       final Drawable background = mBackground;
       ......
       //根据layout过程确定的View位置来设置背景的绘制区域
       if (mBackgroundSizeChanged) {
           background.setBounds(0, 0,  mRight - mLeft, mBottom - mTop);
           mBackgroundSizeChanged = false;
           rebuildOutline();
       }
       ......
           //调用Drawable的draw()方法来完成背景的绘制工作
           background.draw(canvas);
       ......
   }
```

-   **绘制自己(onDraw)**  
    View中onDraw是一个空方法，ViewGroup也没有重新实现。

```cpp
protected void onDraw(Canvas canvas) {
}
```

-   **绘制children(dispatchDraw)**  
    View的dispatchDraw()方法是一个空方法，而且注释说明了如果View包含子类需要重写他。所以ViewGroup肯定重写了，来看看：

```java
   @Override
    protected void dispatchDraw(Canvas canvas) {
        ......
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        ......
        for (int i = 0; i < childrenCount; i++) {
            ......
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        ......
        // Draw any disappearing views that have animations
        if (mDisappearingChildren != null) {
            ......
            for (int i = disappearingCount; i >= 0; i--) {
                ......
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        ......
    }
```

可以看见，ViewGroup确实重写了View的dispatchDraw()方法，该方法内部会遍历每个子View，然后调用drawChild()方法，我们可以看下ViewGroup的drawChild方法，如下：

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

-   **绘制装饰(onDrawScrollBars)**

可以看见其实任何一个View都是有（水平垂直）滚动条的，只是一般情况下没让它显示而已。这部分不做详细分析了。

**draw拓展点:**  
1）View的setWillNotDraw方法：  
如果一个View不需要绘制任何内容，那么设置这个标记位为true后，系统会进行相应的优化，如果需要通过onDraw绘制内容，则需要设置为false。（默认View是false ViewGroup是true）这是个优化手段。

2）区分View动画和ViewGroup布局动画，前者指的是View自身的动画，可以通过setAnimation添加，后者是专门针对ViewGroup显示内部子视图时设置的动画，可以在xml布局文件中对ViewGroup设置layoutAnimation属性（譬如对LinearLayout设置子View在显示时出现逐行、随机、下等显示等不同动画效果）。

3）默认情况下子View的ViewGroup.drawChild绘制顺序和子View被添加的顺序一致，但是你也可以重载ViewGroup.getChildDrawingOrder()方法提供不同顺序。


## 7.3 forceLayout 、invalidate 、requestLayout简述
在之前分析的绘制流程中，我们或多或少都见过这三个方法，他们到底是干什么的，下面做下简单说明：

### 7.3.1 View#forceLayout( )

```csharp
public void forceLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;
}
```

官方描述：强制此视图在下一次布局传递期间进行布局。此方法**不调用父类**的requestLayout()或forceLayout()。

每个View都有个成员变量：mPrivateFlags，在不同的绘制执行路径会对它赋值。在measure方法内部forceLayout用来判断是否执行onMeasure：

```java
final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
```

### 7.3.2 View#invalidate( ) 和 View#postInvalidate( )
invalidate和postInvalidate:都是用来重绘View，区别就是invalidate只能在主线程中调用，postInvalidate可以在子线程中调用.

### 7.3.3 View#requestLayout
requestLayout: 当前view及其以上的viewGroup部分都重新走ViewRootImpl 重新绘制 ，分别重新onMeasure onLayout onDraw ,其中**onDraw比较特殊，有内容变化才会触发**。

最后一张图总结下invalidate/postInvalidate 和 requestLayout

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081557963.png)


## 7.4 参考
[https://blog.csdn.net/yanbober/article/details/46128379](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fyanbober%2Farticle%2Fdetails%2F46128379)  
[https://blog.csdn.net/feiduclear_up/article/details/46772477](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Ffeiduclear_up%2Farticle%2Fdetails%2F46772477)  
[https://www.jianshu.com/p/4a68f9dc8f7c](https://www.jianshu.com/p/4a68f9dc8f7c)  
[https://www.jianshu.com/p/a65861e946cb](https://www.jianshu.com/p/a65861e946cb)  
《Android开发艺术探索》


# 8 Activity、Window、View关系总结
[[0-View概述#ViewRoot、DecorView、Window之间关系]]
本篇文章对之前3篇描述的Activity、Window、View关系做个粗略的总结

在 Activity 创建过程中执行 scheduleLaunchActivity() 之后便调用到了 handleLaunchActivity() 方法。

## 8.1 handleLaunchActivity内调用performLaunchActivity（）
//创建目标Activity对象  
```java
activity = mInstrumentation.newActivity( cl, component.getClassName(), r.intent);
```

## 8.2 执行activity.attach（）

```java
//创建 PhoneWindow  
mWindow = new PhoneWindow(this);

//与activity建立回调关联  
mWindow.setCallback(this);

//设置并获取 WindowManagerImpl 对象  
mWindow.setWindowManager(  
(WindowManager)context.getSystemService(Context.WINDOW_SERVICE),  
mToken, mComponent.flattenToString(),  
(info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);  
mWindowManager = mWindow.getWindowManager();
```

针对WindowManager要多说两句，每个 Activity 会有一个 WindowManager 对象，这个 mWindowManager 就是和 WindowManagerService 进行通信，也是 WindowManagerService 识别 View 具体属于那个 Activity 的关键，创建时传入 IBinder 类型的 mToken。

## 8.3 回调 Activity.onCreate（）

会执行setContentView方法  
```java
installDecor(); 主要就是初始化了DecorView  
mLayoutInflater.inflate(layoutResID, mContentParent);//将layout解析为View树，添加到DecorView的contentView部分
```

这时只是创建了 PhoneWindow，和DecorView，但目前二者也没有任何关系。

## 8.4 WindowManagerGlobal.addView（）

在ActivityThread.performResumeActivity 中，调用 r.activity.performResume()，调用 r.activity.makeVisible（）, makeVisible中：WindowManager 的 addView 的具体实现在 WindowManagerImpl 中.而 WindowManagerImpl 的 addView 又会调用 WindowManagerGlobal.addView()：

```java
public void addView(View view, ViewGroup.LayoutParams params,Display display, Window parentWindow) {  
...  
ViewRootImpl root = new ViewRootImpl(view.getContext(), display);  
view.setLayoutParams(wparams);  
mViews.add(view);  
mRoots.add(root);  
mParams.add(wparams);  
root.setView(view, wparams, panelParentView);  
...  
}

```
一个app进程共享一个WindowManagerGlobal，它是一个统筹大管家，内部方法主要是对View的处理 和 与 WMS的 IPC.

对View的处理交给它的得力助手ViewRootImpl：

## 8.5 ViewRootImpl setView（）

以WindowManagerGlobal的addView为例，最终会调用ViewRootImpl setView（）

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {  
synchronized (this) {  
…  
//开启DecorView绘制流程  
requestLayout();  
...  
//将DecorView添加到window上  
res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,  
getHostVisibility(), mDisplay.getDisplayId(),  
mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,  
mAttachInfo.mOutsets, mInputChannel);  
...  
}  
}
```

两张图总结下：
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081558112.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081559624.png)

总结：  
Activity主要作用还是生命周期的管理，Window是一个视图容器，将Activity与View解耦，WindowManager统一管理View。

所以， Activity与window的关联主要还是体现在生命周期的管理，和key touch事件回调上。 Window与View的关联体现在对View视图的处理上。

## 8.6 参考
[https://blog.csdn.net/freekiteyu/article/details/79408969](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Ffreekiteyu%2Farticle%2Fdetails%2F79408969)
