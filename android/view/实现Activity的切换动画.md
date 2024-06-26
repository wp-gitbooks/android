
## 参考
https://github.com/OCNYang/Android-Animation-Set/wiki/%E5%AE%9E%E7%8E%B0-Activity-%E7%9A%84%E5%88%87%E6%8D%A2%E5%8A%A8%E7%94%BB


## 前引

这里的切换动画指的是 activity 跳转时的动画效果。这里总结了一下，有五种方式实现 activity 切换时实现动画效果。 下面我将依次介绍一下每种实现 activity 切换动画效果的实现方式。

在介绍 activity 的切换动画之前我们先来说明一下实现切换 activity 的两种方式：

-   调用 startActivity 方法启动一个新的 Activity 并跳转其页面
-   调用 finish 方法销毁当前的 Activity 返回上一个 Activity 界面

当调用 startActivity 方法的时候启动一个新的 Activity，这时候就涉及到了旧的 Activity 的退出动画和新的 Activity 的显示动画； 当调用 finish 方法的时候，销毁当前 Activity，就涉及到了当前 Activity 的退出动画和前一个 Activity 的显示动画；

所以我们的 Activity 跳转动画是分为两个部分的：一个 Activity 的销毁动画与一个 Activity 的显示动画。  
明白了这一点之后我们开始看一下第一种实现 Activit y跳转动画的方式： 通过 overridePendingTransition 方法实现 Activity 切换动画。

## [](https://github.com/OCNYang/Android-Animation-Set/wiki/%E5%AE%9E%E7%8E%B0-Activity-%E7%9A%84%E5%88%87%E6%8D%A2%E5%8A%A8%E7%94%BB#%E4%B8%80%E4%BD%BF%E7%94%A8-overridependingtransition-%E6%96%B9%E6%B3%95%E5%AE%9E%E7%8E%B0-activity-%E8%B7%B3%E8%BD%AC%E5%8A%A8%E7%94%BB)一、使用 overridePendingTransition 方法实现 Activity 跳转动画

**overridePendingTransition** 方法是 Activity 中提供的 Activity 跳转动画方法，通过该方法可以实现 Activity 跳转时的动画效果。 下面我们就将通过一个简单的例子看一下如何通过 overridePendingTransition 方法实现 Activity 的切换动画。

Demo 例子中我们实现了 Activity a 中有一个点击按钮，点击按钮实现跳转 Activity b 的逻辑，具体代码如下：

```
/**
 * 点击按钮实现跳转逻辑
 */
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        //在调用了 startActivity 方法之后立即调用 overridePendingTransition 方法
        Intent intent = new Intent(MainActivity.this, SecondActivity.class);
        startActivity(intent);
        overridePendingTransition(R.anim.slide_in_left, R.anim.slide_in_left);
    }
});
```

可以看到我们在调用了 startActivity 方法之后又执行了 overridePendingTransition 方法，而在 overridePendingTransition 方法中传递了两个动画布局文件，我们首先看一下这里的动画文件具体是怎么实现的：

```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:Android="http://schemas.Android.com/apk/res/Android"
     Android:shareInterpolator="false"
     Android:zAdjustment="top">
     <translate
        Android:duration="200"
        Android:fromXDelta="-100.0%p"
        Android:toXDelta="0.0"/>
</set>
```

这里的 overridePendingTransition 方法传递的是两个动画文件 id，第一个参数是需要打开的 Activity 进入时的动画， 第二个参数是需要关闭的 Activity 离开时的动画。这样我们执行了这段代码之后在跳转 Activity 的时候就展示了动画效果：

![overridePendingTransition](https://raw.githubusercontent.com/OCNYang/Android-Animation-Set/master/README_Res/overridePendingTransition.gif?token=AQ83MqQRBPU3KmuAoudHA0i4McHSRGMbks5axHtMwA%3D%3D)

动画的效果是通过 overridePendingTransition 方法实现的，那么下面我们来看一下 overridePendingTransition 方法的定义， 我们在 overridependingTransition 方法在定义的时候有这样的一段注释说明：

```
/**
 * Call immediately after one of the flavors of {@link #startActivity(Intent)}
 * or {@link #finish} to specify an explicit transition animation to
 * perform next.
 *
 * @param enterAnim A resource ID of the animation resource to use for
 * the incoming activity.  Use 0 for no animation.
 * @param exitAnim A resource ID of the animation resource to use for
 * the outgoing activity.  Use 0 for no animation.
 */
public void overridePendingTransition(int enterAnim, int exitAnim) {...}
```

通过这段注释我们能够知道：

-   overridePendingTransition 方法需要在 startActivity 方法或者是 finish 方法调用之后立即执行
-   参数 enterAnim 表示的是从 Activity a 跳转到 Activity b，进入 b 时的动画效果
-   参数 exitAnim 表示的是从 Activity a 跳转到 Activity b，离开 a 时的动过效果
-   若进入 b 或者是离开 a 时不需要动画效果，则可以传值为 0

我们来看一下是不是这样的，首先看一下如果我们在 startActivity 方法调用之后不立即执行 overridePendingTransition 方法， 会有动画效果么？

若我们将 overridePendingTransition 延时 1s 执行呢？

```
/**
 * 点击按钮实现跳转逻辑
 */
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent intent = new Intent(MainActivity.this, SecondActivity.class);
        startActivity(intent);
        overridePendingTransition(R.anim.slide_in_left, R.anim.slide_out_left);
        // 延时1s执行overridePendingTransition方法 
        button1.postDelayed(new Runnable() {
            @Override
            public void run() {
                overridePendingTransition(R.anim.slide_in_top, R.anim.slide_in_top);
            }
        }, 1000);
    }
});
```

执行之后我们能够发现跳转动画没有了，所以 overridePendingTransition 只能在 startActivity 或者是 finish 方法之后执行。

还有一个问题，如果是在 startActivity 之后执行，只是在子线程中执行呢？Activity 的跳转动画能够执行么？

```
/**
 * 点击按钮实现跳转逻辑
 */
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent intent = new Intent(MainActivity.this, SecondActivity.class);
        startActivity(intent);
        overridePendingTransition(R.anim.slide_in_left, R.anim.slide_out_left);
        /**
         * 在子线程中执行overridePendingTransition方法
         */
        new Thread(new Runnable() {
            @Override
            public void run() {
                overridePendingTransition(R.anim.slide_in_left, R.anim.slide_out_left);
            }
        }).start();

    }
});
```

测试后发现 Activity 的切换效果还是执行，也就是说 overridePendingTransition 方法也是可以在子线程中执行的， 当然这并没什么卵用。

好吧，在介绍完了使用 overridePendingTransition 方法实现 Activity 切换动画之后我们下面看一下使用 style 的方式定义实现 Activity 的切换动画。

## [](https://github.com/OCNYang/Android-Animation-Set/wiki/%E5%AE%9E%E7%8E%B0-Activity-%E7%9A%84%E5%88%87%E6%8D%A2%E5%8A%A8%E7%94%BB#%E4%BA%8C%E4%BD%BF%E7%94%A8-style-%E7%9A%84%E6%96%B9%E5%BC%8F%E5%AE%9A%E4%B9%89-activity-%E7%9A%84%E5%88%87%E6%8D%A2%E5%8A%A8%E7%94%BB)二、使用 style 的方式定义 Activity 的切换动画

**1_ 定义 Application 的 style**

```
<!-- 系统Application定义 -->
<application
    Android:allowBackup="true"
    Android:icon="@mipmap/ic_launcher"
    Android:label="@string/app_name"
    Android:supportsRtl="true"
    Android:theme="@style/AppTheme">
```

**2_ 定义具体的 AppTheme 样式**

其中这里的 `windowAnimationStyle` 就是我们定义 Activity 切换动画的 style。而 `@anim/slide_in_top` 就是我们定义 的动画文件，也就是说通过为 Application 设置 style，然后为 windowAnimationStyle 设置动画文件就可以全局的为 Activity 的跳转配置动画效果。

```
<!-- Base application theme. -->
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <item name="colorPrimary">@color/colorPrimary</item>
    <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
    <item name="colorAccent">@color/colorAccent</item>
    <item name="Android:windowAnimationStyle">@style/activityAnim</item>
</style>

<!-- 使用style方式定义activity切换动画 -->
<style name="activityAnim">
    <item name="Android:activityOpenEnterAnimation">@anim/slide_in_top</item>
    <item name="Android:activityOpenExitAnimation">@anim/slide_in_top</item>
</style>
```

而在 windowAnimationStyle 中存在四种动画：

-   activityOpenEnterAnimation // 用于设置打开新的 Activity 并进入新的 Activity 展示的动画
-   activityOpenExitAnimation // 用于设置打开新的 Activity 并销毁之前的 Activity 展示的动画
-   activityCloseEnterAnimation // 用于设置关闭当前 Activity 进入上一个 Activity 展示的动画
-   activityCloseExitAnimation // 用于设置关闭当前 Activity 时展示的动画

**3_ 测试代码，实现 Activity 切换操作**

```
/**
 * 点击按钮,实现Activity的跳转操作
 * 通过定义style的方式实现activity的跳转动画
 */
button2.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        /**
         * 普通的Intent跳转Activity实现
         */
        Intent intent = new Intent(MainActivity.this, SecondActivity.class);
        startActivity(intent);
    }
});
```

这时候我们我们执行 diamante 逻辑之后就能发现 Activity 在切换的时候出现了动画效果，说明我们设置的 style 起作用了。

## [](https://github.com/OCNYang/Android-Animation-Set/wiki/%E5%AE%9E%E7%8E%B0-Activity-%E7%9A%84%E5%88%87%E6%8D%A2%E5%8A%A8%E7%94%BB#%E4%B8%89%E4%BD%BF%E7%94%A8-activityoptions-%E5%88%87%E6%8D%A2%E5%8A%A8%E7%94%BB%E5%AE%9E%E7%8E%B0-activity-%E8%B7%B3%E8%BD%AC%E5%8A%A8%E7%94%BB)三、使用 ActivityOptions 切换动画实现 Activity 跳转动画

## [](https://github.com/OCNYang/Android-Animation-Set/wiki/%E5%AE%9E%E7%8E%B0-Activity-%E7%9A%84%E5%88%87%E6%8D%A2%E5%8A%A8%E7%94%BB#%E5%9B%9B%E4%BD%BF%E7%94%A8-activityoptions-%E4%B9%8B%E5%90%8E%E5%86%85%E7%BD%AE%E7%9A%84%E5%8A%A8%E7%94%BB%E6%95%88%E6%9E%9C%E9%80%9A%E8%BF%87-style-%E7%9A%84%E6%96%B9%E5%BC%8F)四、使用 ActivityOptions 之后内置的动画效果通过 style 的方式

## [](https://github.com/OCNYang/Android-Animation-Set/wiki/%E5%AE%9E%E7%8E%B0-Activity-%E7%9A%84%E5%88%87%E6%8D%A2%E5%8A%A8%E7%94%BB#%E4%BA%94%E4%BD%BF%E7%94%A8-activityoptions-%E5%8A%A8%E7%94%BB%E5%85%B1%E4%BA%AB%E7%BB%84%E4%BB%B6%E7%9A%84%E6%96%B9%E5%BC%8F%E5%AE%9E%E7%8E%B0%E8%B7%B3%E8%BD%AC-activity-%E5%8A%A8%E7%94%BB)五、使用 ActivityOptions 动画共享组件的方式实现跳转 Activity 动画

以上三、四、五、皆属于 转场动画范畴，  
请到 [**《Ⅵ. Transition Animation / 转场动画 & 共享元素》**](https://github.com/OCNYang/Android-Animation-Set/tree/master/transition-animation) 查看详细讲解。

**附录**  
摘录自：[原文地址](https://blog.csdn.net/qq_23547831/article/details/51821159)