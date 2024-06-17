# 概述

## 背景    [[插件加载机制]]
2015年是Android插件化技术突飞猛进的一年，随着业务的发展各大厂商都碰到了Android Native平台的瓶颈：

1.  从技术上讲，业务逻辑的复杂导致代码量急剧膨胀，各大厂商陆续出到65535方法数的天花板；同时，运营为王的时代对于模块热更新提出了更高的要求。
2.  在业务层面上，功能模块的解耦以及维护团队的分离也是大势所趋；各个团队维护着同一个App的不同模块，如果每个模块升级新功能都需要对整个app进行升级，那么发布流程不仅复杂而且效率低下；在讲究小步快跑和持续迭代的移动互联网必将遭到淘汰。

H5和Hybird可以解决这些问题，但是始终比不上native的用户体验；于是，国外的FaceBook推出了`react-native`；而国内各大厂商几乎都选择纯native的插件化技术。可以说，Android的未来必将是`react-native`和插件化的天下。

`react-native`资料很多，但是讲述插件化的却凤毛菱角；插件化技术听起来高深莫测，实际上要解决的就是两个问题：

**1.  代码加载**
**2.  资源加载**


### 代码加载
类的加载可以使用Java的`ClassLoader`机制，但是对于Android来说，并不是说类加载进来就可以用了，很多组件都是有“生命”的；因此对于这些有血有肉的类，必须给它们注入活力，也就是所谓的**组件生命周期管理**；

另外，如何管理加载进来的类也是一个问题。假设多个插件依赖了相同的类，是抽取公共依赖进行管理还是插件单独依赖？这就是**ClassLoader的管理问题**；



### 资源加载 
资源加载方案大家使用的原理都差不多，都是用`AssetManager`的隐藏方法`addAssetPath`；但是，不同插件的资源如何管理？是公用一套资源还是插件独立资源？共用资源如何避免资源冲突？对于资源加载，有的方案共用一套资源并采用资源分段机制解决冲突（要么修改`aapt`要么添加编译插件）；有的方案选择独立资源，不同插件管理自己的资源。

目前国内开源的较成熟的插件方案有[DL](https://github.com/singwhatiwanna/dynamic-load-apk)和[DroidPlugin](https://github.com/Qihoo360/DroidPlugin)；但是DL方案仅仅对Frameworl的表层做了处理，严重依赖`that`语法，编写插件代码和主程序代码需单独区分；而DroidPlugin通过Hook增强了Framework层的很多系统服务，开发插件就跟开发独立app差不多；就拿Activity生命周期的管理来说，DL的代理方式就像是牵线木偶，插件只不过是操纵傀儡而已；而DroidPlugin则是借尸还魂，插件是有血有肉的系统管理的真正组件；DroidPlugin Hook了系统几乎所有的Sevice，欺骗了大部分的系统API；掌握这个Hook过程需要掌握很多系统原理，因此学习DroidPlugin对于整个Android FrameWork层大有裨益。



## 是什么？用途？


# 实现

## 已知知识-代理 [[静态代理和动态代理]]

## 代理hook
我们知道代理有比原始对象更强大的能力，比如飞到国外买东西，比如坑钱坑货；那么很自然，如果我们自己创建代理对象，然后把原始对象替换为我们的代理对象，那么就可以在这个代理对象为所欲为了；修改参数，替换返回值，我们称之为Hook。

下面我们Hook掉`startActivity`这个方法，使得每次调用这个方法之前输出一条日志；（当然，这个输入日志有点点弱，只是为了展示原理；只要你想，你想可以替换参数，拦截这个`startActivity`过程，使得调用它导致启动某个别的Activity，指鹿为马！）

首先我们得找到被Hook的对象，我称之为Hook点；什么样的对象比较好Hook呢？自然是**容易找到的对象**。什么样的对象容易找到？**静态变量和单例**；在一个进程之内，静态变量和单例变量是相对不容易发生变化的，因此非常容易定位，而普通的对象则要么无法标志，要么容易改变。我们根据这个原则找到所谓的Hook点。

然后我们分析一下`startActivity`的调用链，找出合适的Hook点。我们知道对于`Context.startActivity`（Activity.startActivity的调用链与之不同），由于`Context`的实现实际上是`ContextImpl`;我们看`ConetxtImpl`类的`startActivity`方法：

```java
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();
    if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                + " Is this really what you want?");
    }
    mMainThread.getInstrumentation().execStartActivity(
        getOuterContext(), mMainThread.getApplicationThread(), null,
        (Activity)null, intent, -1, options);
}
```

这里，实际上使用了`ActivityThread`类的`mInstrumentation`成员的`execStartActivity`方法；注意到，`ActivityThread` 实际上是主线程，而主线程一个进程只有一个，因此这里是一个良好的Hook点。

接下来就是想要Hook掉我们的主线程对象，也就是把这个主线程对象里面的`mInstrumentation`给替换成我们修改过的代理对象；要替换主线程对象里面的字段，首先我们得拿到主线程对象的引用，如何获取呢？`ActivityThread`类里面有一个静态方法`currentActivityThread`可以帮助我们拿到这个对象类；但是`ActivityThread`是一个隐藏类，我们需要用反射去获取，代码如下：

```java
// 先获取到当前的ActivityThread对象
Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
currentActivityThreadMethod.setAccessible(true);
Object currentActivityThread = currentActivityThreadMethod.invoke(null);
```

拿到这个`currentActivityThread`之后，我们需要修改它的`mInstrumentation`这个字段为我们的代理对象，我们先实现这个代理对象，由于JDK动态代理只支持接口，而这个`Instrumentation`是一个类，没办法，我们只有手动写静态代理类，覆盖掉原始的方法即可。（`cglib`可以做到基于类的动态代理，这里先不介绍）

```java
public class EvilInstrumentation extends Instrumentation {

    private static final String TAG = "EvilInstrumentation";

    // ActivityThread中原始的对象, 保存起来
    Instrumentation mBase;

    public EvilInstrumentation(Instrumentation base) {
        mBase = base;
    }

    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {

        // Hook之前, XXX到此一游!
        Log.d(TAG, "\n执行了startActivity, 参数如下: \n" + "who = [" + who + "], " +
                "\ncontextThread = [" + contextThread + "], \ntoken = [" + token + "], " +
                "\ntarget = [" + target + "], \nintent = [" + intent +
                "], \nrequestCode = [" + requestCode + "], \noptions = [" + options + "]");

        // 开始调用原始的方法, 调不调用随你,但是不调用的话, 所有的startActivity都失效了.
        // 由于这个方法是隐藏的,因此需要使用反射调用;首先找到这个方法
        try {
            Method execStartActivity = Instrumentation.class.getDeclaredMethod(
                    "execStartActivity",
                    Context.class, IBinder.class, IBinder.class, Activity.class, 
                    Intent.class, int.class, Bundle.class);
            execStartActivity.setAccessible(true);
            return (ActivityResult) execStartActivity.invoke(mBase, who, 
                    contextThread, token, target, intent, requestCode, options);
        } catch (Exception e) {
            // 某该死的rom修改了  需要手动适配
            throw new RuntimeException("do not support!!! pls adapt it");
        }
    }
}
```

Ok，有了代理对象，我们要做的就是偷梁换柱！代码比较简单，采用反射直接修改：

```java
public static void attachContext() throws Exception{
    // 先获取到当前的ActivityThread对象
    Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
    Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
    currentActivityThreadMethod.setAccessible(true);
    Object currentActivityThread = currentActivityThreadMethod.invoke(null);

    // 拿到原始的 mInstrumentation字段
    Field mInstrumentationField = activityThreadClass.getDeclaredField("mInstrumentation");
    mInstrumentationField.setAccessible(true);
    Instrumentation mInstrumentation = (Instrumentation) mInstrumentationField.get(currentActivityThread);

    // 创建代理对象
    Instrumentation evilInstrumentation = new EvilInstrumentation(mInstrumentation);

    // 偷梁换柱
    mInstrumentationField.set(currentActivityThread, evilInstrumentation);
}
```

好了，我们启动一个Activity测试一下，结果如下：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210720203836.png)


### 总结
可见，Hook确实成功了！这就是使用代理进行Hook的原理——偷梁换柱。整个Hook过程简要总结如下：

1. 寻找Hook点，原则是**静态变量或者单例对象**，尽量Hook pulic的对象和方法，非public不保证每个版本都一样，需要适配。
2. 选择合适的代理方式，如果是接口可以用动态代理；如果是类可以手动写代理也可以使用cglib。
3. 偷梁换柱——用代理对象替换原始对象


# 应用
##  Binder Hook [[Hook机制之BinderHook]]

## AMS&PMS [[Hook机制之AMS&PMS]]



# hook机制  [[1-Android插件化开发指南#hook]]

https://cloud.tencent.com/developer/article/1329477

http://weishu.me/2016/02/16/understand-plugin-framework-binder-hook/

http://weishu.me/2016/01/28/understand-plugin-framework-overview/

http://weishu.me/2016/01/28/understand-plugin-framework-proxy-hook/



# 参考
http://weishu.me/2016/01/28/understand-plugin-framework-proxy-hook/



# 面试题

hook技术用过么（面试官说插件化技术生命周期很难管理）

- [weishu.me/2016/01/28/…](http://weishu.me/2016/01/28/understand-plugin-framework-proxy-hook/)
- Activity启动
  - Activity启动时，AMS会在Activity启动时检查清单文件中activity是否注册，如果没注册就会抛异常
  - 如果AMS不抛出异常，就可以不注册也能启动了，所以需要hook
- Hook技术：
  - Hook是什么？Hook中文意思就是钩子。简单说，它的**作用就是改变代码的正常执行流程**。
  - 实现hook的技术：反射和动态代理
- Hook点：钩子挂的地方就是Hook点。
- 查找Hook点的原则：
  - 1.尽量静态变量或者单例对象。原因是静态变量一般不会修改
  - 2.尽量Hook publit的对象和方法。原因是public一般是提供对外调用的方法，谷歌工程师一般不会去修改
