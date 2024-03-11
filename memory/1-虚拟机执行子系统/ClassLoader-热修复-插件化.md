# 热修复的应用场景

热修复就是在APP上线以后，如果突然发现有缺陷了，如果重新走发布流程可能时间比较长，重新安装APP用户体验也不会太好；热修复就是通过发布一个插件，使APP运行的时候加载插件里面的代码，从而解决缺陷，并且对于用户来说是无感的（用户也可能需要重启一下APP）。

# 热修复的原理

先说结论吧，就是将补丁 dex 文件放到 dexElements 数组靠前位置，这样在加载 class 时，优先找到补丁包中的 dex 文件，加载到 class 之后就不再寻找，从而原来的 apk 文件中同名的类就不会再使用，从而达到修复的目的

理解这个原理，需要了解一下Android的代码加载的机制；

## Android运行流程

简单来讲整体流程是这样的：
1、Android程序编译的时候，会将.java文件编译时.class文件
2、然后将.class文件打包为.dex文件
3、然后Android程序运行的时候，Android的Dalvik/ART虚拟机就加载.dex文件
4、加载其中的.class文件到内存中来使用

## 类加载器

负责加载这些.class文件的就是类加载器（ClassLoader），APP启动的时候，会创建一个自己的ClassLoader实例，我们可以通过下面的代码拿到当前的ClassLoader

```
ClassLoader classLoader = getClassLoader();
Log.i(TAG, "[onCreate] classLoader" + ":" + classLoader.toString());
```

ClassLoader加载类的方法就是loadClass可以看一下源码，是通过双亲委派模型（Parents Delegation Model），它首先不会自己去尝试加载这个类， 而是把这个请求委派给父类加载器去完成，当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类） 时， 子加载器才会尝试自己去完成加载，最后是调用自己的findClass方法完成的

```
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
```

ClassLoader是一个抽象类，通过打印可以看出来当前的ClassLoader是一个PathClassLoader；看一下PathClassLoader的构造函数，可以看出，需要传入一个dexPath也就是dex包的路径，和父类加载器；

```
    //dexPath 包含 dex 的 jar 文件或 apk 文件的路径集，多个以文件分隔符分隔，默认是“：”    public PathClassLoader(String dexPath, ClassLoader parent) {        super((String)null, (File)null, (String)null, (ClassLoader)null);        throw new RuntimeException("Stub!");    }
```

PathClassLoader是BaseDexClassLoader的子类，除此之外BaseDexClassLoader还有一个子类是DexClassLoader，optimizedDirectory用来缓存优化的 dex 文件的路径，即从 apk 或 jar 文件中提取出来的 dex 文件；

```
    //dexPath 包含 dex 的 jar 文件或 apk 文件的路径集，多个以文件分隔符分隔，默认是“：”
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }
```

这两个的区别，网上的答案是

> 1、DexClassLoader可以加载jar/apk/dex，可以从SD卡中加载未安装的apk
> 2、PathClassLoader只能加载系统中已经安装过的apk

从这个答案可以知道，我们想要加载更新的插件，肯定是使用 DexClassLoader；但是有点离谱的是其实我用两个都能成功，也许我加载的插件包名这些都和原APP一致导致的吧。

### 类加载器的运行流程

具体的实现都在BaseDexClassLoader里面，看一下里面的实现（源码看不了，网上搜一下），下面是一个构造方法

```
public BaseDexClassLoader(String dexPath, File optimizedDirectory, String libraryPath, ClassLoader parent) {
    super(parent);
    this.originalPath = dexPath;
    this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
}
```

构造方法创建了一个DexPathLis，里面解析了dex文件的路径，并将解析的dex文件都存在this.dexElements里面

```
public DexPathList(ClassLoader definingContext, String dexPath, String libraryPath, File optimizedDirectory) {
…
    //将解析的dex文件都存在this.dexElements里面
    this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory);
}
 //解析dex文件
private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory) {
    ArrayList<Element> elements = new ArrayList<Element>();
    for (File file : files) {
        ZipFile zip = null;
        DexFile dex = null;
        String name = file.getName();
        if (name.endsWith(DEX_SUFFIX)) {
            dex = loadDexFile(file, optimizedDirectory);
        } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX) || name.endsWith(ZIP_SUFFIX)) {
            zip = new ZipFile(file);
        }
        ……
        if ((zip != null) || (dex != null)) {
            elements.add(new Element(file, zip, dex));
        }
    } return elements.toArray(new Element[elements.size()]);
}
```

然后我们再回头看一下ClassLoader加载类的方法,就是loadClass()，最后调用findClass方法完成的;BaseDexClassLoader 重写了该方法，如下

```
 @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        // 使用pathList对象查找name类
        Class c = pathList.findClass(name, suppressedExceptions);
        return c;
    }
```

最终是调用 pathList的findClass方法，看一下方法如下

```
public Class findClass(String name, List<Throwable> suppressed) {
    // 遍历从dexPath查询到的dex和资源Element
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        // 如果当前的Element是dex文件元素
        if (dex != null) {
            // 使用DexFile.loadClassBinaryName加载类
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}
```

### 结论

所以整个类加载流程就是

> 1、类加载器BaseDexClassLoader先将dex文件解析放到pathList到dexElements里面
> 2、加载类的时候从dexElements里面去遍历，看哪个dex里面有这个类就去加载，生成class对象

所以我们可以将自己的dex文件加载到dexElements里面，并且放在前面，加载的时候就可以加载我们插件中的类，不会加载后面的,从而替换掉原来的class。

# 热修复的实现

知道了原理，实现就比较简单了，就添加新的dex对象到当前APP的ClassLoader对象（也就是BaseDexClassLoader）的pathList里面的dexElements；要添加就要先创建，我们使用DexClassLoader先加载插件，先生成插件的dexElements，然后再添加就好了。

当然整个过程需要使用反射来实现。除此以外，常用的两种方法是使用apk作为插件和使用dex文件作为插件；下面的两个实现都是对程序中的一个方法进行了修改，然后分别打了 dex包和apk包，程序运行起来执行的方法就是插件里面的方法而不是程序本身的方法；

## dex插件

对于dex文件作为插件，和之前说的流程完全一致，先将修改了的类进行打包成dex包，将dex进行加载，插入到dexElements集合的前面即可；打包流程是先将.java文件编译成.class文件，然后使用SDK工具打包成dex文件人，然后APP下载，加载即可；

### dex打包工具

d8 作为独立工具纳入了 Android 构建工具 28.0.1 及更高版本中：`C:\Users\hanpei\AppData\Local\Android\Sdk\build-tools\29.0.2\d8.bat`；输入字节码可以是 *.class 文件或容器（例如 JAR、APK 或 ZIP 文件）的任意组合。您还可以添加 DEX 文件作为 d8 的输入，以将这些文件合并到 DEX 输出中

```
 d8 MyProject/app/build/intermediates/classes/debug/*/*.class
```

### 具体的代码实现

代码的注释已经很详细了，就不再进行说明了

```
//在Application中进行替换
public class MApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        //dex作为插件进行加载
        dexPlugin();
    }
    ...
  /**
     * dex作为插件加载
     */
    private void dexPlugin(){
        //插件包文件
        File file = new File("/sdcard/FixTest.dex");
        if (!file.exists()) {
            Log.i("MApplication", "插件包不在");
            return;
        }
        try {
            //获取到 BaseDexClassLoader 的  pathList字段
            // private final DexPathList pathList;
            Field pathListField = BaseDexClassLoader.class.getDeclaredField("pathList");
            //破坏封装，设置为可以调用
            pathListField.setAccessible(true);
            //拿到当前ClassLoader的pathList对象
            Object pathListObj = pathListField.get(getClassLoader());
            //获取当前ClassLoader的pathList对象的字节码文件（DexPathList ）
            Class<?> dexPathListClass = pathListObj.getClass();
            //拿到DexPathList 的 dexElements字段
            // private final Element[] dexElements；
            Field dexElementsField = dexPathListClass.getDeclaredField("dexElements");
            //破坏封装，设置为可以调用
            dexElementsField.setAccessible(true);
            //使用插件创建 ClassLoader
            DexClassLoader pathClassLoader = new DexClassLoader(file.getPath(), getCacheDir().getAbsolutePath(), null, getClassLoader());
            //拿到插件的DexClassLoader 的 pathList对象
            Object newPathListObj = pathListField.get(pathClassLoader);
            //拿到插件的pathList对象的 dexElements变量
            Object newDexElementsObj = dexElementsField.get(newPathListObj);
            //拿到当前的pathList对象的 dexElements变量
            Object dexElementsObj=dexElementsField.get(pathListObj);
            int oldLength = Array.getLength(dexElementsObj);
            int newLength = Array.getLength(newDexElementsObj);
            //创建一个dexElements对象
            Object concatDexElementsObject = Array.newInstance(dexElementsObj.getClass().getComponentType(), oldLength + newLength);
            //先添加新的dex添加到dexElement
            for (int i = 0; i < newLength; i++) {
                Array.set(concatDexElementsObject, i, Array.get(newDexElementsObj, i));
            }
            //再添加之前的dex添加到dexElement
            for (int i = 0; i < oldLength; i++) {
                Array.set(concatDexElementsObject, newLength + i, Array.get(dexElementsObj, i));
            }
            //将组建出来的对象设置给 当前ClassLoader的pathList对象
            dexElementsField.set(pathListObj, concatDexElementsObject);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

## apk插件

apk作为插件，就是我们重新打了一个新的apk包作为插件，打包很简单方便，缺点就是文件大；使用apk的话就没必要是将dex插入dexElements里面去，直接将之前的dexElements替换就可以了；

### 具体的实现

代码的注释已经很详细了，就不再进行说明了

```
/**
     * apk作为插件加载
     */
    private void apkPlugin() {
        //插件包文件
        File file = new File("/sdcard/FixTest.apk");
        if (!file.exists()) {
            Log.i("MApplication", "插件包不在");
            return;
        }
        try {
            //获取到 BaseDexClassLoader 的  pathList字段
            // private final DexPathList pathList;
            Field pathListField = BaseDexClassLoader.class.getDeclaredField("pathList");
            //破坏封装，设置为可以调用
            pathListField.setAccessible(true);
            //拿到当前ClassLoader的pathList对象
            Object pathListObj = pathListField.get(getClassLoader());
            //获取当前ClassLoader的pathList对象的字节码文件（DexPathList ）
            Class<?> dexPathListClass = pathListObj.getClass();
            //拿到DexPathList 的 dexElements字段
            // private final Element[] dexElements；
            Field dexElementsField = dexPathListClass.getDeclaredField("dexElements");
            //破坏封装，设置为可以调用
            dexElementsField.setAccessible(true);
            //使用插件创建 ClassLoader
            DexClassLoader pathClassLoader = new DexClassLoader(file.getPath(), getCacheDir().getAbsolutePath(), null, getClassLoader());
            //拿到插件的DexClassLoader 的 pathList对象
            Object newPathListObj = pathListField.get(pathClassLoader);
            //拿到插件的pathList对象的 dexElements变量
            Object newDexElementsObj = dexElementsField.get(newPathListObj);
            //将插件的 dexElements对象设置给 当前ClassLoader的pathList对象
            dexElementsField.set(pathListObj, newDexElementsObj);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```



# 插件化方案

看到这里，你应该大概理解了`classloader`加载流程，其实`java`这层的`classloader`代码量并不多，主要集中在`c`层，但是我们在`java`层进行`hook`便可实现热修复。
 结合网上的资料及源码的阅读一共有三种方案

## 方案1:向`dexElements`进行插入新的`dex`（目前最常见的方式）

从上面的`ClassLoader#loadClass`方法你就会知道，初始化的时候会进入`BaseDexClassLoader#findClass`方法中通过遍历`dexElements`进行查找`dex`文件，因为`dexElements`是一个数组，所以我们可以通过反射的形式，将需要热修复的`dex`文件插入到数组`首部`，这样遍历数组的时候就会优先读取你插入的`dex`，从而实现热修复。

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210809142747.png)



### DexClassLoader不是允许你加载外部dex吗？用DexClassLoader#loadClass不就行了

我们知道`DexClassLoader`是允许你加载外部`dex`文件的，所以网上有一些例子介绍通过`DexClassLoader#loadClass`可以加载到你的`dex`文件中的方法，那么有一些网友就会有疑问，我直接通过调用`DexClassLoader#loadClass`去获取我传入的外部`dex`文件中的`class`，不就行了，这样确实是可以的，但是它仅适用于新增的类，而不能去替换旧的类，因为通过上面的`dexElements`数组的生成以及`委派双亲机制`，你就会知道它的父类是先去把你应用类组装进来，当你调用`DexClassLoader`去`loadClass`时，是先委派父类去`loadClass`，如果查找不到才会到子类自行查找，也就是说应用中本来就已经存在`B.class`了，那么父类`loadClass`会直接返回，而你真正需要返回的其实是子类中的`B.class`，所以才说只适用于新增的类，你不通过一些手段修改源码层，是无法实现替换类的。

## 方案2:在ActivityThread中替换LoadedApk的mClassLoader对象

小编在开发MPlugin的时候，使用了下面的方法，但发现当你插件apk中进行跳转的下一个页面的时候，若引了第三方的库，会抛出无法载入该第三方库控件异常。
 实现代码如下：



```dart
  public static void loadApkClassLoader(Context context,DexClassLoader dLoader){
        try{
            // 配置动态加载环境
            Object currentActivityThread = RefInvoke.invokeStaticMethod(
                    "android.app.ActivityThread", "currentActivityThread",
                    new Class[] {}, new Object[] {});//获取主线程对象 http://blog.csdn.net/myarrow/article/details/14223493
            String packageName = context.getPackageName();//当前apk的包名
            ArrayMap mPackages = (ArrayMap) RefInvoke.getFieldOjbect(
                    "android.app.ActivityThread", currentActivityThread,
                    "mPackages");
            WeakReference wr = (WeakReference) mPackages.get(packageName);
            RefInvoke.setFieldOjbect("android.app.LoadedApk", "mClassLoader",
                    wr.get(), dLoader);
        }catch(Exception e){
             e.printStackTrace();
        }
    }
```

## 方案3:通过自定义ClassLoader实现class拦截替换

我们知道`PathClassLoader`是加载已安装的`apk`的`dex`，那我们可以
 在 `PathClassLoader` 和 `BootClassLoader` 之间插入一个 自定义的`MyClassLoader`，而我们通过`ClassLoader#loadClass`方法中的第`2`步知道，若`parent`不为空，会调用`parent.loadClass`方法，固我们可以在`MyClassLoader`中重写`loadClass`方法，在这个里面做一个判断去拦截替换掉我们需要修复的`class`。

### 如何拿到我们需要修复的class呢？

我当时首先想到的是通过`DexClassLoader`直接去`loadClass`来获得需要热修复的`Class`，但是通过`ClassLoader#loadClass`方法分析，可以知道加载查找`class`的第`1`步是调用`findLoadedClass`，这个方法主要作用是检查该类是否被加载过，如果加载过则直接返回，所以如果你想通过`DexClassLoader`直接去`loadClass`来获得你需要热修复的`Class`，是不可能完成替换的（热修复），因为你调用`DexClassLoader.loadClass`已经属于首次加载了，那么意味着下次加载就直接在`findLoadedClass`方法中返回`class`了，是不会再往下走，从而`MyClassLoader#loadClass`方法也不可能会被回调，也就无法实现修复。
 通过`BaseDexClassLoader#findClass`方法你就会知道，这个方法在父`ClassLoader`不能加载该类的时候才由自己去加载，我们可以通过这个方法来获得我们的`class`，因为你调用这个方法的话，是不会被缓存起来。也就不存在`ClassLoader#loadClass`中的第`1`步就查找到就被返回。

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210809142909.png)



### 方案3代码

```dart
public class HookUtil {
    /**
     * 在 PathClassLoader 和 BootClassLoader 之间插入一个 自定义的MyClassLoader
     * @param classLoader
     * @param newParent
     */
    public static void injectParent(ClassLoader classLoader, ClassLoader newParent) {
        try {
            Field parentField = ClassLoader.class.getDeclaredField("parent");
            parentField.setAccessible(true);
            parentField.set(classLoader, newParent);
        } catch (IllegalArgumentException e) {
            throw new RuntimeException(e);
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     * 反射调用findClass方法获取dex中的class类
     * @param context
     * @param dexPath
     * @param className
     */
    public static void hookFindClass(Context context,String dexPath,String className){
        DexClassLoader dexClassLoader = new DexClassLoader(dexPath, context.getDir("dex",context.MODE_PRIVATE).getAbsolutePath(),null, context.getClassLoader());
        try {
            Class<?> herosClass = dexClassLoader.getClass().getSuperclass();
            Method m1 = herosClass.getDeclaredMethod("findClass", String.class);
            m1.setAccessible(true);
            Class newClass = (Class) m1.invoke(dexClassLoader, className);
            ClassLoader pathClassLoader = MyApplication.getContext().getClassLoader();
            MyClassLoader myClassLoader = new MyClassLoader(pathClassLoader.getParent());
            myClassLoader.registerClass(className, newClass);
            injectParent(pathClassLoader, myClassLoader);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}
```



```java
public class MyClassLoader extends ClassLoader {

    public Map<String,Class> myclassMap;

    public MyClassLoader(ClassLoader parent) {
        super(parent);
        myclassMap = new HashMap<>();
    }

    /**
     * 注册类名以及对应的类
     * @param className
     * @param myclass
     */
    public void registerClass(String className,Class myclass){
        myclassMap.put(className,myclass);
    }

    /**
     * 移除对应的类
     * @param className
     */
    public void removeClass(String className){
        myclassMap.remove(className);
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        Class myclass = myclassMap.get(name);
        //重写父类loadClass方法，实现拦截
        if(myclass!=null){
            return myclass;
        }else{
            return super.loadClass(name, resolve);
        }
    }
}
```

## 关于CLASS_ISPREVERIFIED标记

因为在 `Dalvik`虚拟机下，执行 `dexopt` 时，会对类进行扫描，如果类里面所有直接依赖的类都在同一个 dex 文件中，那么这个类就会被打上 `CLASS_ISPREVERIFIED` 标记，如果一个类有 `CLASS_ISPREVERIFIED`标记，那么在热修复时，它加载了其他 dex 文件中的类，会报经典的

`Class ref in pre-verified class resolved to unexpected implementation`异常
 通过源码搜索并没有找到`CLASS_ISPREVERIFIED`标记这个关键词，通过在`android7.0、8.0`上进行热修复，也没有遇到这个异常，猜测这个问题只属于android5.0以前（关于解决方法网上有很多，本文就不讲述了），因为android5.0后新增了`art`。



## 最后

看到这里相信java层的ClassLoader机制你已经熟悉得差不多了，相对于插件化而言你已经前进了一步，但仍有一些问题需要去思考解决的，比如解决资源加载、混淆、加壳等问题，为了更好的完善热修复机制，你也可以去阅读下c层的逻辑，尽管热修复带来了很多便利，但个人也并不是太认同热修复的使用，毕竟是通过hook去修改源码层，因为a





# 参考

https://jaeger.itscoder.com/android/2016/09/20/nuva-source-code-analysis.html

https://www.zybuluo.com/Tyhj/note/1762634

http://weishu.me/2016/04/05/understand-plugin-framework-classloader/

https://www.jianshu.com/p/95387cc07e3c