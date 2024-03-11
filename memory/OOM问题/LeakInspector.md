## LeakInspector

LeakInspector 是腾讯内部的使用的 **一站式内存泄漏解决方案**，它是 Android 手机经过长期积累和提炼、**集内存泄漏检测、自动修复系统Bug、自动回收已泄露Activity内资源、自动分析GC链、白名单过滤** 等功能于一体，并 **深度对接研发流程、自动分析责任人并提缺陷单的全链路体系**。

### 那么，LeakInspector 与 LeakCanary 又有什么不同之处呢？

它们之间主要有 **四个方面** 的不同，如下所示：

#### 一、检测能力与原理方面不同

##### 1、检测能力

它们都支持对 Activity、Fragment 及其它自定义类的泄漏检测，但是，LeakInspector 还 **增加了 Btiamp 的检测能力**，如下所示：

- 1）、检测有没有在 View 上 decode 超过该 View 尺寸的图片，若有则上报出现问题的 Activity 及与其对应的 View id，并记录它的个数与平均占用内存的大小。
- 2）、检测图片尺寸是否超过所有手机屏幕大小，违规则报警。

这一个部分的实现原理，我们可以采用 ARTHook 的方式来实现，还不清楚的朋友请再仔细看看大图检测的部分。

##### 2、检测原理

两个工具的泄漏检测原理都是在 onDestroy 时检查弱引用，**不同之处在于 LeakInspector 直接使用 WeakReference 来检测对象是否已经被释放**，而 LeakCanary 则使用 ReferenceQueue，两者效果是一样的。

并且针对 Activity，我们通常都会使用 Application的 registerActivityLifecycleCallbacks 来注册 Activity 的生命周期，以重写 onActivityDestroyed 方法实现。但是在 **Android 4.0 以下**，系统并没有提供这个方法，为了避免手动在每一个 Activity 的 onDestroy 中去添加这份代码，我们可以使用 **反射 Instrumentation 来截获 onDestory**，以降低接入成本。代码如下所示：

```
Class<?> clazz = Class.forName("android.app.ActivityThread");
Method method = clazz.getDeclaredMethod("currentActivityThread", null);
method.setAccessible(true);
sCurrentActivityThread = method.invoke(null, null);
Field field = sCurrentActivityThread.getClass().getDeclaredField("mInstumentation");
field.setAccessible(true);
field.set(sCurrentActivityThread, new MonitorInstumentation());
复制代码
```

#### 二、泄漏现场处理方面不同

##### 1、dump 采集

两者都能采集 dump，但是 LeakInspector 提供了**回调方法**，我们可以**增加更多的自定义信息**，如运行时 Log、trace、dumpsys meminfo 等信息，以辅助分析定位问题。

##### 2、白名单定义

这里的白名单是为了处理一些系统引起的泄漏问题，以及一些因为 **业务逻辑要开后门的情形而设置** 的。分析时如果碰到白名单上标识的类，则不对这个泄漏做后续的处理。二者的配置差异有如下两点：

- 1）、LeakInspector 的白名单以 XML 配置的形式存放在服务器上。
  - 优点：跟产品甚至不同版本的应用绑定，我们可以很方便地修改相应的配置。
  - 缺点：白名单里的类不区分系统版本一刀切。
- 1）、而LeakCanary的白名单是直接写死在其源码的AndroidExcludedRefs类里。
  - 优点：定义非常详细，并区分系统版本。
  - 缺点：每次修改必定得重新编译。
- 2）、LeakCanary 的系统白名单里定义的类比 LeakInspector 中定义的多很多，因为它没有自动修复系统泄漏功能。

##### 3、自动修复系统泄漏

针对系统泄漏，LeakInspector 通过 **反射自动修复** 了目前碰到的一些系统泄漏，只要在 **onDestory** 里面 **调用** 一个修复系统泄漏的方法即可。而 LeakCanary 虽然能识别系统泄漏，但是它仅仅对该类问题给出了分析，没有提供实际可用的解决方案。

##### 4、回收资源（Activity内存泄漏兜底处理）

如果检测到发生了内存泄漏，LeakInspector 会对整个 Activity 的 View 进行遍历，把图片资源等一些占内存的数据释放掉，保证此次泄漏只会泄漏一个Activity的空壳，尽量减少对内存的影响。代码大致如下所示：

```
if (View instanceof ImageView) {
    // ImageView ImageButton处理
    recycleImageView(app, (ImageView) view);
} else if (view instanceof TextView) {
    // 释放TextView、Button周边图片资源
    recycleTextView((TextView) view);
} else if (View instanceof ProgressBar) {
    recycleProgressBar((ProgressBar) view);
} else {
    if (view instancof android.widget.ListView) {
        recycleListView((android.widget.ListView) view);
    } else if (view instanceof android.support.v7.widget.RecyclerView) {
        recycleRecyclerView((android.support.v7.widget.RecyclerView) view);
    } else if (view instanceof FrameLayout) {
        recycleFrameLayout((FrameLayout) view);
    } else if (view instanceof LinearLayout) {
        recycleLinearLayout((LinearLayout) view);
    }
    
    if (view instanceof ViewGroup) {
        recycleViewGroup(app, (ViewGroup) view);
    }
}
复制代码
```

这里以 recycleTextView 为例，它回收资源的方式如下所示：

```
private static void recycleTextView(TextView tv) {
    Drawable[] ds = tv.getCompoundDrawables();
    for (Drawable d : ds) {
        if (d != null) {
            d.setCallback(null);
        }
    }
    tv.setCompoundDrawables(null, null, null, null);
    // 取消焦点，让Editor$Blink这个Runnable不再被post，解决内存泄漏。
    tv.setCursorVisible(false);
}
复制代码
```


