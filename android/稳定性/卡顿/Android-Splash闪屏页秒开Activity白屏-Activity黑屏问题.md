# 线索
图片、动画
每个启动方法耗时：adb shell am start -W  .....   然后通过 Systrace 和 Profile 进行处理。


今天在测试应用的时候发现，在进入闪屏页之前，会有1-2秒的白屏或者黑屏（颜色与设置的主题有关）。

# 一、本文章主要解决：

1.  APP启动时白屏/黑屏、Activity打开时白屏/黑屏。
2.  APP启动速度慢，如何实现点击ICON后APP秒开。APP启动加速。

# 二、了解绘制整个窗口：

1.  绘制背景。
2.  绘制View本身的内容。
3.  绘制子View。
4.  绘制修饰内容（例如滚动条)。

闪屏原因剖析`StartingWindow`（Preview Window）

# 三、常见的做法：

我们正常开发中会在`Activity的onCreate()`方法中调用`setContentView(View)`设置该`Activity`的显示布局。

当打开一个`Activity`时，如果这个`Activity`所属`Application`还没有在运行，系统会为这个Activity的创建一个进程（每开启一个进程都会有一个`Application`，所以`Application的onCreate()`可能会被调用多次），但进程的创建与初始化都需要时间，在这个动作完成之前，如果初始化的时间过长，屏幕上可能没有任何动静，用户会以为没有点到按钮。所以既不能停在原来的地方又没到显示新的界面，怎么办呢？这就有了`StartingWindow`（也称之为`PreviewWindow`）的出现，这样看起来就像Activity已经启动起来了，只是数据内容还没有初始化好。

`StartingWindow`一般出现在应用程序进程创建并初始化成功前，所以它是个临时窗口，对应的`WindowType`是`TYPE_APPLICATION_STARTING`。目的是告诉用户，系统已经接受到操作，正在响应，在程序初始化完成后实现目的UI，同时移除这个窗口。

`StartingWindow`就是我们要讨论的白屏和黑屏的“元凶”，一般情况下我们会对`Application`和`Activity`设置`Theme`，系统会根据设置的Theme初始化`StartingWindow`。`Window`布局的顶层是`DecorView`，`StartingWindow`显示一个空`DecorView`，但是会给这个`DecorView`应用这个Activity指定的Theme，如果这个`Activity`没有指定`Theme`就用`Application`的（`Application`系统要求必须设置`Theme`）。

# 解决方案

既然决定解决这个问题，那么从哪里入手呢，Android在选择展示黑屏或者白屏的时候，是根据你**设定的主题**而不同的，也就是说，虽然你的代码没有被执行，你的配置文件却被提前读取了，用来作为展示Preview Window界面的依据。

所以，我们的解决方案的切入口就是整个APP的manifest文件，更确切的说应该是主题配置文件。

## 方案一 ：开历史倒车

这个方案就是禁止加载Preview Window，具体做法如下：

style.xml

```xml
<style name="APPTheme" parent="@android:style/Theme.Holo.NoActionBar">
   <item name="android:windowDisablePreview">true</item>
</style>
```

将APPTheme设定为启动的Activity的主题，即可禁止Preview Window，当然，也有人通过把preview window设置为全透明，也达成了类似的效果。

结果就是，当你点击APP时，界面会无响应一段时间，然后进入APP。

我个人强烈**不推荐**这么做，因为Android想方设法提升的用户体验一下子被你打回解放前。

---

## 方案二：图片-自定义Preview Window

具体方法如下：

style.xlm

```xml
<style name="APPTheme" parent="@android:style/Theme.Holo.NoActionBar">
    <item name="android:windowBackground">@drawable/splash_icon</item>
</style>
```

同样将主题设置到启动的Activity的主题中，windowBackground就是即将展示的preview window。其中splash_icon可以是一整张图片，网上很多小伙伴也都是这么做的。其实它也可以是一个能解析出图片资源的XML文件，好像只有layer-list这种能做得到,因为它能够将多个drawable叠加起来展示。

splash_icon.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:opacity="opaque">
    <item android:drawable="@color/white"/>
    <item>
        <bitmap
            android:gravity="center"
            android:src="@drawable/qq"/>
    </item>
</layer-list>
```

这样设置之后，当你点击APP，会立马进入你配置的界面，然后启动欢迎页，效果如下

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303231522170.gif)

那么，将preview window直接设置为图片和设置为xml文件有什么区别或者优劣呢？我先卖个关子。先谈谈这种方案的优劣，首先这种方案已经解决了原生preview window的单调难看的问题，在原来的基础上进一步提升了用户体验。可是我们的APP都是有欢（guang）迎（gao）页的，从preview window跳转到欢（guang）迎（gao）页是不可避免的，这样的话，两个界面的切换就会显得很突兀的，

所以强迫症的我们，尝试让这两个界面的切换变成一个界面的变化，从而进一步提升显示效果，怎么样才能让两个界面切换看起来像是在同一个界面里的变化呢？答案就是： 动画。

在这种需求下，图片和xml文件的区别就出来了，因为**后者可以帮助我们更准确的实现动画。**

---

## 方案三：动画-自定义Preview Window增强版

废话少说，我们先来看效果

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303231522171.gif)

  
有了动画之后，界面切换顺畅了许多。

上面的动画实现其实非常简单，无非就是放缩，移动，渐变的组合使用（我仅仅用作范例给大家参考），具体的动画代码细节就不谈了，有兴趣可以[去github上看本次项目的demo](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fladingwu%2FSplash "https://github.com/ladingwu/Splash")，我们重点来聊一聊思路。

在这里我们需要明确一点的是，preview window只能是静态图，它本身是不展示动画的，我们这里的动画，其实是在进入欢迎页之后的展示的。明确了这一点之后，整个动画效果的实现思路其实就已经摆在我们眼前了，那就是当界面从 **Preview Window** 跳转到 **欢迎页** 的时候，欢迎页必须首先展示一个和**Preview Window**一模一样的界面，让人看起来好像界面还没切换一样，然后再慢慢切换到欢迎页。

然后，我们再来谈谈为什么设置xml的方式可以帮助我们更准确的实现动画，就是因为要保证**Preview Window**和**欢迎页**最开始展示的界面保持绝对一致，只有通过xml的布局才是达到这种效果。

好了，启动页做到这个份儿上，应该就可以交货了，不过还有一个小问题需要大家注意的，那就是我们给Preview Window设置的背景图如果不做处理，图片就会一直存在于内存中，所以，当我们进入到欢迎页的时候，不要忘了把背景图设置为空：

SplashActivity.java

```java
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        //将window的背景图设置为空
        getWindow().setBackgroundDrawable(null);
        super.onCreate(savedInstanceState);
    }
```

到这里，关于Android启动页的相关问题就都讲完了。


# 四、解决办法

## 闪屏页Activity设置透明的主题

```xml
<style name="SplashTheme" parent="AppTheme">
    <item name="android:windowFullscreen">true</item>
    <item name="android:windowIsTranslucent">true</item>
</style>

```

如上设置后`APP`和`Activity`启动时，我们的`StartingWindow`会应用我们这个透明背景的主题，跳转时确实没有白屏和黑屏了，但是这样设置会产生如下后果： 

1.1、给`SplashActivity`设置后，用户点击我们`APP`图标后，需要等待2秒左右的时候才会显示`contentView`。造成了`APP`启动速度慢的假象，其实`Activity`已经启动了，只是`background`是透明的，这时候你点击桌面的其他地方是无效的。这样就和`Google`的初衷背道而驰了，所以还要继续往下看。 

1.2、给其他`Activity`设置后，会导致通过`overridePendingTransition`设置的启动关闭`Activity`的动画无效。需要在`style`中重新写如下几个动画：

```xml
<style name="AppTheme" parent="AppBaseTheme">
<item name="android:windowAnimationStyle">@style/Animation.Activity.Translucent.Style</item>
<item name="android:windowFullscreen">true...
<item name="android:windowIsTranslucent">true...
</style>

<style name="Animation.Activity.Style" parent="@android:style/Animation.Activity">
<item name="android:activityOpenEnterAnimation">...
<item name="android:activityOpenExitAnimation">...
<item name="android:activityCloseEnterAnimation">...
<item name="android:activityCloseExitAnimation">...
</style>

<style name="Animation.Activity.Translucent.Style" parent="@android:style/Animation.Translucent"> 
<item name="android:windowEnterAnimation">...
<item name="android:windowExitAnimation">...
</style>

```

1.3、`Activity`之间的跳转可能偶尔会看到桌面一闪而过（如果`SplashActivity`和其他`Activity`都设置了透明）。

小结：**一般情况下是只会给SplashActivity设置一个透明背景的主题，其他Activity不会设置，经过实践，这种体验是最好的。但是如果要做到APP秒开还是不行的，和我们的文章开头所分析的原理相斥了**。

## 实现秒开
2.实现秒开的效果 我们之前设置了`Window`透明，实现了去掉白屏和黑屏，现在要弄一个颜色或者图片来代替白屏和黑屏，所以首先要把原来`style` 中的透明属性去掉。然后给`Window`设置一个背景颜色或者图片。

实现步骤 
2.1、首先在res/drawable下新建一个`layer-list`，名字随便取，比如`splash.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 背景颜色 -->
    <item android:drawable="@color/white" />

    <item>
        <!-- 图片 -->
        <bitmap
            android:gravity="center"
            android:src="@drawable/wel_page" />
    </item>
</layer-list>

```

`layer-list`大家都会写吧，上面是背景颜色，下面是一张图，这张图可以是全屏的图，可以是一张小图。如果是全屏的图，那上面的颜色也可以不用设置，如果是小图，就要指定下颜色了，并且可以指定图片在位置。

2.2 给window设置背景

```xml
<style name="SplashTheme" parent="AppBaseTheme">
    <!-- 欢迎页背景引用刚才写好的 -->
    <item name="android:windowBackground">@drawable/splash</item>
    <item name="android:windowFullscreen">true</item>
    <!-- <item name="android:windowIsTranslucent">true</item> --> <!-- 透明背景不要了 -->
</style>

```

上面的`<item name="android:windowBackground">`可以用我们上面的`layer-lis`t作为背景，当然也可以设置个全屏的图片。

2.3、在`AndroidManifest.xml`中定义`SplashActivity`的`theme`为`SplashTheme`：

```xml
<activity android:name=".SplashActivity"
    android:theme="@style/SplashTheme">
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
</activity>
```

2.4、`SplashActivity`的实现，在`onCreate()`启动你的`MainActivity`即可，其他什么都别干：

```java
public class SplashActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        startActivity(new Intent(this, MainActivity.class));
        finish();
    }
}
```
