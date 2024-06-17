# 参考

https://www.jianshu.com/p/d52960e3f6f2



# 线索

调用流程为： **Application 的 attachBaseContext ---> ContentProvider 的 onCreate ----> Application 的 onCreate ---> Activity、Service 等的 onCreate**(Activity 和 Service 不分先后)



# 背景

在做一个项目时，我们想在应用最早启动时，先做一些判断，然后根据判断的结果再决定要不要对其他应用提供服务。

对其他应用提供服务指的是，我们的应用中有 ContentProvider，第三方应用通过 call 方法调用到我们提供的 ContentProvider，ContentProvider 执行逻辑后并给调用的返回结果。当第三方应用调用我们的应用时，我们的应用存在启动和未启动的两种情况。

刚开始，我们将判断逻辑写在了自定义的 Application 的 onCreate 方法中，但等到测试时发现了很多意想不到的情况，比如：

逻辑判断之后的结果是不给第三方应用提供“服务”，但有时候第三方应用能够使用服务，而有时候第三方应用不能使等等的问题。

于是我们跟踪代码，发现了 四大组件 以及 Application 的各个方法( attachBaseContext、onCreate、call 等)启动顺序，跟我们之前理解的稍稍不一样。

在弄清楚了 四大组件 和 Application 在应用冷启动时的执行顺序后，我们才把遇到的问题彻底解决。

# 验证试验

为了测试 四大组件 和 Application 的各种方法( attachBaseContext、onCreate、call 等)被系统调用的顺序，我们创建一个简单的应用，只包含这5个组件，不考虑一个应用多进程的情况，代码分别为：

![img](https:////upload-images.jianshu.io/upload_images/2203721-1a8c58982edc79cd?imageMogr2/auto-orient/strip|imageView2/2/w/605/format/webp)



***MainApplication.java***

![img](https:////upload-images.jianshu.io/upload_images/2203721-68546f8bd3836577?imageMogr2/auto-orient/strip|imageView2/2/w/426/format/webp)



***MainActivity.java\***

![img](https:////upload-images.jianshu.io/upload_images/2203721-dc4b253bf836f731?imageMogr2/auto-orient/strip|imageView2/2/w/530/format/webp)



***MainService.java***

![img](https:////upload-images.jianshu.io/upload_images/2203721-29e3eddece8c558d?imageMogr2/auto-orient/strip|imageView2/2/w/448/format/webp)



***MainReceiver.java***

![img](https:////upload-images.jianshu.io/upload_images/2203721-c96419148b8e9b4e?imageMogr2/auto-orient/strip|imageView2/2/w/503/format/webp)



***MainProvider.java***

![img](https:////upload-images.jianshu.io/upload_images/2203721-b953175fb40f751f?imageMogr2/auto-orient/strip|imageView2/2/w/576/format/webp)



在以下几个场景测试时，均已冷启动的方式启动应用。

冷启动，指的是在系统没有创建apk这个进程时启动apk。

注意在测试的手机上，不要让测试的应用被禁止关联启动或自启动：

## **场景一**，点击桌面的图标启动应用，日志如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-7714cc11c9c834ae?imageMogr2/auto-orient/strip|imageView2/2/w/508/format/webp)



## **场景二**，通过另外一个应用以启动Service的形式启动应用，其中启动 MainService 的代码如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-bcbf78457e5c4c76?imageMogr2/auto-orient/strip|imageView2/2/w/475/format/webp)



日志如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-c609c2aa1be31c9d?imageMogr2/auto-orient/strip|imageView2/2/w/513/format/webp)



## **场景三**，应用通过接受开机广播启动的方式启动，日志如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-4743d9a08a5f912f?imageMogr2/auto-orient/strip|imageView2/2/w/512/format/webp)



## **场景四**，其他应用调用 ContentProvider 的 call 方法启动，其中，调用 MainProvider 的 call 代码如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-49eccd2fe37fa583?imageMogr2/auto-orient/strip|imageView2/2/w/465/format/webp)



日志如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-1555d5741a4802ed?imageMogr2/auto-orient/strip|imageView2/2/w/519/format/webp)



# 结论：

从上面四个场景可以看出：

**1.**Application 的 attachBaseContext 方法是优先执行的；

**2.**ContentProvider 的 onCreate 的方法比 Application 的 onCreate 的方法先执行；

**3.**Activity、Service的 onCreate 方法以及 BroadcastReceiver 的 onReceive 方法，是在 MainApplication 的 onCreate 方法之后执行的；

4.调用流程为： Application 的 attachBaseContext ---> ContentProvider 的 onCreate ----> Application 的 onCreate ---> Activity、Service 等的 onCreate(Activity 和 Service 不分先后)

问题

## **问题一：**ContentProvider 的 onCreate 一定是优先于 Application 的 onCreate 执行的吗？

为了验证这个问题，MainApplication 的代码不变，我们将 MainProvider 的 onCreate 的代码改为：

![img](https:////upload-images.jianshu.io/upload_images/2203721-920a1da6487562c6?imageMogr2/auto-orient/strip|imageView2/2/w/441/format/webp)



我们再在上面第四种场景上进行验证，日志如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-c72cb9424ee1d78a?imageMogr2/auto-orient/strip|imageView2/2/w/512/format/webp)



**问题一结论：**

确实是在 ContentProvider 的 onCreate 执行完成之后，才会执行 Application 的 onCreate 的。

## **问题二：**ContentProvider中 的 call方法 是在 Application 的 onCreate 执行完之后才执行的吗？

为了验证这个问题，我们将 MainProvider 和 MainApplication 的代码改为：

![img](https:////upload-images.jianshu.io/upload_images/2203721-29eabea232d70a3c?imageMogr2/auto-orient/strip|imageView2/2/w/463/format/webp)



![img](https:////upload-images.jianshu.io/upload_images/2203721-074bdfecf5769dc1?imageMogr2/auto-orient/strip|imageView2/2/w/498/format/webp)



我们还在第四个场景下验证，日志如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-465bfad8067310fe?imageMogr2/auto-orient/strip|imageView2/2/w/510/format/webp)



从日志中可以发现，Application 的 onCreate 执行时，ContentProvider 的 call方法 也在同时执行。

**问题二结论：**

Application 的 onCreate方法 和 Provider 的 call方法**不是顺序执行，而是会同时执行**。

## **问题三：**有比 Application 的 attachBaseContext方法 更早执行的方法吗？

有，比如：Application所在类的构造方法。为了验证这个问题，将代码改为：

![img](https:////upload-images.jianshu.io/upload_images/2203721-5321c41bb90a6a90?imageMogr2/auto-orient/strip|imageView2/2/w/563/format/webp)



程序启动后，日志为：

![img](https:////upload-images.jianshu.io/upload_images/2203721-c465dfe8994e4cbd?imageMogr2/auto-orient/strip|imageView2/2/w/509/format/webp)



**问题三结论：**

Application 的构造方法早于 Application 的 attachBaseContext方法 调用。

那么有没有比 Application 的构造方法还早被调用的方法呢？有，自己可以再想想哦。

遇到的坑

好了，我们知道 attachBaseContext 的方法在“一般情况下是最早执行的“，那么在项目中为了能”尽早“的提前初始化某些模块、功能或者参数，那么我们会把代码从 Application 的onCreate方法 提前到 attachBaseContext方法 中。嗯，一切感觉起来那么美好，直到你运行程序崩溃时...

好吧好吧，那些“坑”终于还是来了。

**“坑”一：**在 Application 的 attachBaseContext方法 中，使用了 getApplicationContext方法。

当我发现在 attachBaseContext方法 中使用 getApplicationContext方法 返回**null**时，内心是崩溃。

所以，如果在 attachBaseContext方法 中要使用 context 的话，那么使用**this**吧，别再使用 getApplicationContext() 方法了。下文有分析为什么。

**“坑”二：**这个其实不算很坑，也不会引起崩溃，但需要注意：

在 Application 的 attachBaseContext方法 中，去调用自身的 ContentProvider，那么这个 ContentProvider 会被初始化两次，也就是说这个 ContentProvider 会被两次调用到onCreate。如果你在 ContentProvider 的 onCreate 中有一些逻辑，那么一定要检查是否会有影响。

做一下验证，在 Application 中调用 Provider 的 call方法，并在 MainActivity 中的 onCreate方法 中调用 Provider 的 call方法，Application 的代码，Provider 的代码，Activity 的代码分别如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-c7cb0d6597654ffc?imageMogr2/auto-orient/strip|imageView2/2/w/519/format/webp)



启动应用后，日志如下：

![img](https:////upload-images.jianshu.io/upload_images/2203721-554ffcb38321089c?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)



可以看到，MainProvider 的 onCreate 的方法被调用了两次（因为 MainProvider 的两次 onCreate 打印出的自身对象不一样），而在 MainActivity 中调用到 call方法 执行的类，跟 MainApplication 在 attachBaseContext方法 执行的类是同一个。

源码分析

好了，现象、问题和“坑”都经历了一遍，那么 为什么 会是这样呢？

我们通过看源码，来跟踪：

**1.**Application 的 attachBaseContext、ContentProvider 的 onCreate 以及 Application 的 onCreate 在源码中的调用顺序。

**2.**为什么在 Application 的 attachBaseContext 中调用 getApplicationContext 得到的是**null**。

先看第一个问题，我们知道Java进程的入口方法一般都是在main中，而Android为了让应用开发者不需要关心应用的创建，已经帮我们进行封装，其实应用 main方法 是在 ActivityThread.java 中的。

我们查看 ActivityThread.java 的源码，本文以下的源码都以**6.0.1_r10**基础。

**a. ActivityThread.java 的 main方法：**

![img](https:////upload-images.jianshu.io/upload_images/2203721-0b7a9d1bc1cd98f2?imageMogr2/auto-orient/strip|imageView2/2/w/583/format/webp)



**b. ActivityThread.java 的 attach方法：**

![img](https:////upload-images.jianshu.io/upload_images/2203721-32a3cf317b180aa5?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)



**c. ActivityThread.java 的 handleBindApplication(AppBindData data)方法：**

![img](https:////upload-images.jianshu.io/upload_images/2203721-9f380a106162a4e6?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)



**d. LoaderApk.java 的 makeApplication方法**

![img](https:////upload-images.jianshu.io/upload_images/2203721-c23416982d7500c6?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)



**e. Instrumentation.java的相关方法**

![img](https:////upload-images.jianshu.io/upload_images/2203721-8df6d33a0d64a46f?imageMogr2/auto-orient/strip|imageView2/2/w/632/format/webp)



**f. Application.java 的 attach方法**

![img](https:////upload-images.jianshu.io/upload_images/2203721-26db1bb3daf35397?imageMogr2/auto-orient/strip|imageView2/2/w/449/format/webp)



**g. ActivityThread.java 的 handleBindApplication方法：**

![img](https:////upload-images.jianshu.io/upload_images/2203721-428aa8c973a17ee9?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)



**h. 继续跟踪 installContentProviders 这个方法，而这个方法是会调用到 installProvider方法 中的，还是在 ActivityThread.java 中：**

![img](https:////upload-images.jianshu.io/upload_images/2203721-993ff6802540dcd8?imageMogr2/auto-orient/strip|imageView2/2/w/599/format/webp)



**i. 看 ContentProvider.java 中的 attachInfo方法(frameworks/base/core/java/android/content/ContentProvider.java)**

![img](https:////upload-images.jianshu.io/upload_images/2203721-a83a45cfd183fbad?imageMogr2/auto-orient/strip|imageView2/2/w/604/format/webp)



**j. 关于 注释7 的 mInstrumentation.callApplicationOnCreate(app) 调用到的 Instrumentation.java 中的方法**

![img](https:////upload-images.jianshu.io/upload_images/2203721-5e8607a4e7f1dd87?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)



**看第二问题，为什么在我们自定义 Application 中的 attachBaseContext方法 中，调用 getApplicationContext() 为 null 呢？**

**1. 跟踪 getApplicationContext() 发现是在 ContextWrapper.java 中实现的：**

![img](https:////upload-images.jianshu.io/upload_images/2203721-67c2c69ed77457de?imageMogr2/auto-orient/strip|imageView2/2/w/635/format/webp)



**2. 我们看 ContextImpl 的 getApplicationContext方法：**

![img](https:////upload-images.jianshu.io/upload_images/2203721-d6d26a4ac27046c2?imageMogr2/auto-orient/strip|imageView2/2/w/639/format/webp)



**3. mPackageInfo 是什么时候赋值的呢？我们从 ContextImpl 实例化的地方入手，在注释 5.1 之前的一行代码看到了 ContextImpl 的实例化代码，跟进代码发现，果不其然，看到了 mPackageInfo 被赋值的地方：**

![img](https:////upload-images.jianshu.io/upload_images/2203721-5e366ae6cd062c04?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)



**4. 注释b.1所在的流程  早于 注释5.4 的（在注释5.4时才调用到了Application的attachBaseContext方法），在我们自定义的Application中attachBaseContext调用getApplicationContext方法时，就到了注释b，而此时mPackageInfo是不为空的，所以会执行mPackageInfo.getApplication()，那么我们再看一下LoadedApk.java中的getApplication方法（正如前面所说，mPackageInfo是LoadedApk的实例），**

![img](https:////upload-images.jianshu.io/upload_images/2203721-adc93a7ddfcec170?imageMogr2/auto-orient/strip|imageView2/2/w/544/format/webp)



![img](https:////upload-images.jianshu.io/upload_images/2203721-c6d09c7940bf2073?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)



看到这里找到原因所在了：

因为我们在 Application 的 attachBaseContext方法 中调用 getApplicationContext() 时， mApplication 还没有被赋值，所以返回的是空，只有把 attachBaseContext方法 执行完成后，mApplication 才会被赋值。

附图一张：

![img](https:////upload-images.jianshu.io/upload_images/2203721-9b79f731473f05bd?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)



