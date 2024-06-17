# 缘起

利用 ContentProvider 来初始化你的 Library， 这个相信大家已经不太陌生了，下面简要说下。

# 1利用 ContentProvider 初始化 Library:

```
在日常开发过程中， 经常会遇到 Library 都需要传入 Context 参数以完成初始化，此时这个 Context 参数一般会从 Application 对象的 onCreate 方法中获取。于是，很多 library 都会提供一个 init 方法，在Application Object中完成调用.
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        Library.init(this);
    }
}
```

至此， 相信大家可以看到依赖第三方 library 的时候，你必须得写一个自定义的 Application， 这样的用户体验可不太好。从 Android Debug Database， Firebase 中受到启发， 我们可以在 library 中创建一个 ContentProvider 来， 然后在 Manifest 中完成必要的注册信息， 在应用 Application 起来的时候会自动去初始化 library 中定义的 ContentProvider，现在就可以从 ContentProvider 中的 onCreate 里面得到 context 来初始化自己的 library。

```
public class FirstProvider extends ContentProvider {
    @Override
    public void attachInfo(Context context, ProviderInfo info) {
        super.attachInfo(context, info);
    }
    @Override
    public boolean onCreate() {
        context = getContext();
        // 这里可以初始化你需要的代码
        return true;
    }
    ......
}
​``` 
这里顺便说一下Application， ContentProvider 初始化的顺序：

​```java
Application.attachBaseContext(super before) -> Application.attachBaseContext(super after) -> ContentProvider.attachInfo(super before) -> ContentProvider.onCreate() -> ContentProvider.attachInfo(super after) -> Application.onCreate(super before) -> Application.onCreate(super after)
```

# 2自定义 ContentProvider 初始化顺序：

1. 如果 library 中有业务需要用到多个自定义 ContentProvider，如 A、B、C， 但是自定义用来初始化的 provider 为D， 由于初始化流程在 provider D 中开始的， 那么 A、B、C 就必须在 D 之后起来才行，那么D ->A、B、C 顺序怎么来定义呢？也就是如何保证 D 最先初始化。

2. 为了定义 provider 的初始化顺序，可以再 Manifest 中设置 **initOrder** 的值(值越大，最先初始化)，同时如果 Library 中有多进程， 那么也需要设置 android:multiprocess，具体如下：

   ```
   <provider
   android:authorities="com.sivan.DContentProvider"
               android:exported="false"
               android:multiprocess="true"
               android:initOrder="100"
               android:name=".DContentProvider" />
   ```

如此在 Demo 中定义了 FirstProvider，SecondProvider 以及 ThirdProvider 的 initOrder 的值：

```
<provider
        android:authorities="com.sivan.FirstProvider"
        android:exported="false"
        android:initOrder="100"
        android:name=".FirstProvider" />

<provider
        android:authorities="com.sivan.SecondProvider"
        android:exported="false"
        android:initOrder="50"
        android:name=".SecondProvider" />

<provider
        android:authorities="com.sivan.ThirdProvider"
        android:exported="false"
        android:name=".ThirdProvider" />
```

最后得到的日志信息如下：

```
MyApplication: attachBaseContext 0 pid = 3725
MyApplication: attachBaseContext 1 pid = 3725
FirstProvider: attachInfo 0 com.sivan.FirstProvider
FirstProvider: onCreate pid =3725
FirstProvider: attachInfo 1 com.sivan.FirstProvider
SecondProvider: attachInfo 0 info = com.sivan.SecondProvider
SecondProvider: onCreate  pid = 3725
SecondProvider: attachInfo 1 info = com.sivan.SecondProvider
ThirdProvider: attachInfo 0 info = com.sivan.ThirdProvider
ThirdProvider: onCreate  pid = 3725
ThirdProvider: attachInfo 1 info = com.sivan.ThirdProvider
MyApplication: onCreate 0 pid = 3725
MyApplication: onCreate 1 pid = 3725
```



可以看出初始化流程：FirstProvider -> SecondProvider -> ThirdProvider

所以我们需要根据 android:initOrder 来调整自定义用来初始化的 ContentProvider， 要保证 D 在 A、B、C 之前来初始化。