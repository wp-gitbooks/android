# 概述

## 什么是ClassLoader

程序在启动的时候，并不会一次性加载程序所要用的所有class文件，而是根据程序的需要，通过Java的类加载机制（ClassLoader）来动态加载某个class文件到内存当中的，从而只有class文件被载入到了内存之后，才能被其它class所引用。所以**ClassLoader就是用来动态加载class文件到内存当中用**的。



## Java默认提供的三个ClassLoader

**负责加载JDK中的核心类库，如：rt.jar、resources.jar、charsets.jar等**，可通过如下程序获得该类加载器从哪些地方加载了相关的jar或class文件：



## ClassLoader的体系架构

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210806221346.png)





# 原理-ClassLoader加载类的原理

## 介绍
​    ClassLoader使用的是**双亲委托模型** 来搜索类的，每个ClassLoader实例都有一个父类加载器的引用（不是继承的关系，是一个包含的关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其它ClassLoader实例的的父类加载器。当一个ClassLoader实例需要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。否则将这个找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。


## 为什么要使用双亲委托这种模型呢？
​    因为这样可以避免重复加载，当父亲已经加载了该类的时候，就没有必要子ClassLoader再加载一次。考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的String来动态替代java核心api中定义的类型，这样会存在非常大的安全隐患，而双亲委托的方式，就可以避免这种情况，因为String已经在启动时就被引导类加载器（Bootstrcp ClassLoader）加载，所以用户自定义的ClassLoader永远也无法加载一个自己写的String，除非你改变JDK中ClassLoader搜索类的默认算法。



## JVM在搜索类的时候，如何判定两个class是相同的呢？
   JVM在判定两个class是否相同时，不仅要判断**两个类名是否相同，而且要判断是否由同一个类加载器实例加载**的。只有两者同时满足的情况下，JVM才认为这两个class是相同的。就算两个class是同一份class字节码，如果被两个不同的ClassLoader实例所加载，JVM也会认为它们是两个不同class。比如网络上的一个Java类org.classloader.simple.NetClassLoaderSimple，javac编译之后生成字节码文件NetClassLoaderSimple.class，ClassLoaderA和ClassLoaderB这两个类加载器并读取了NetClassLoaderSimple.class文件，并分别定义出了java.lang.Class实例来表示这个类，对于JVM来说，它们是两个不同的实例对象，但它们确实是同一份字节码文件，如果试图将这个Class实例生成具体的对象进行转换时，就会抛运行时异常java.lang.ClassCaseException，提示这是两个不同的类型。现在通过实例来验证上述所描述的是否正确：



# 重要API

`ClassLoader` 里面有三个重要的方法 `loadClass()`、`findClass()` 和 `defineClass()`。`loadClass()` 方法是加载目标类的入口，它首先会查找当前 `ClassLoader` 以及它的双亲里面是否已经加载了目标类，如果没有找到就会让双亲尝试加载，
如果双亲都加载不了，就会调用 `findClass()` 让自定义加载器自己来加载目标类。`ClassLoader` 的 `findClass()` 方法是需要子类来覆盖的，
不同的加载器将使用不同的逻辑来获取目标类的字节码。拿到这个字节码之后再调用 `defineClass()` 方法将字节码转换成 `Class` 对象

## loadClass



## findClass


## defineClass



# 自定义ClassLoader-自定义加载器

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210806221452.png)



**示例：自定义一个Network的ClassLoader，用于加载网络上的class文件**



```
package classloader;  
  
import java.io.ByteArrayOutputStream;  
import java.io.InputStream;  
import java.net.URL;  
  
/** 
 * 加载网络class的ClassLoader 
 */  
public class NetworkClassLoader extends ClassLoader {  
      
    private String rootUrl;  
  
    public NetworkClassLoader(String rootUrl) {  
        this.rootUrl = rootUrl;  
    }  
  
    @Override  
    protected Class<?> findClass(String name) throws ClassNotFoundException {  
        Class clazz = null;//this.findLoadedClass(name); // 父类已加载     
        //if (clazz == null) {  //检查该类是否已被加载过  
            byte[] classData = getClassData(name);  //根据类的二进制名称,获得该class文件的字节码数组  
            if (classData == null) {  
                throw new ClassNotFoundException();  
            }  
            clazz = defineClass(name, classData, 0, classData.length);  //将class的字节码数组转换成Class类的实例  
        //}   
        return clazz;  
    }  
  
    private byte[] getClassData(String name) {  
        InputStream is = null;  
        try {  
            String path = classNameToPath(name);  
            URL url = new URL(path);  
            byte[] buff = new byte[1024*4];  
            int len = -1;  
            is = url.openStream();  
            ByteArrayOutputStream baos = new ByteArrayOutputStream();  
            while((len = is.read(buff)) != -1) {  
                baos.write(buff,0,len);  
            }  
            return baos.toByteArray();  
        } catch (Exception e) {  
            e.printStackTrace();  
        } finally {  
            if (is != null) {  
               try {  
                  is.close();  
               } catch(IOException e) {  
                  e.printStackTrace();  
               }  
            }  
        }  
        return null;  
    }  
  
    private String classNameToPath(String name) {  
        return rootUrl + "/" + name.replace(".", "/") + ".class";  
    }  
  
}  

```



测试类

```
package classloader;  
  
public class ClassLoaderTest {  
  
    public static void main(String[] args) {  
        try {  
            /*ClassLoader loader = ClassLoaderTest.class.getClassLoader();  //获得ClassLoaderTest这个类的类加载器 
            while(loader != null) { 
                System.out.println(loader); 
                loader = loader.getParent();    //获得父加载器的引用 
            } 
            System.out.println(loader);*/  
              
  
            String rootUrl = "http://localhost:8080/httpweb/classes";  
            NetworkClassLoader networkClassLoader = new NetworkClassLoader(rootUrl);  
            String classname = "org.classloader.simple.NetClassLoaderTest";  
            Class clazz = networkClassLoader.loadClass(classname);  
            System.out.println(clazz.getClassLoader());  
              
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
      
} 
```



打印结果：

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210806221649.png)


# Android中ClassLoader

## 一、Android 中的 dex 文件

Java程序中，JVM虚拟机是通过类加载器ClassLoader加载.jar文件里面的类的。Android也类似，不过Android用的是Dalvik/ART虚拟机，不是JVM，也不能直接加载.jar文件，而是加载dex文件。

Google 使用了自己的 Dalvik 来运行应用，所以class 不能在 AndroidDalvik 的 java 环境中运行， android 的 class 文件实际上只是编译过程中的中间目标文件，需要链接成 dex 文件后才能在 dalvik 上运行。

- Android 应用打包成 apk 文件时，class 文件会被打包成一个或者多个 dex 文件。将一个 apk 文件后缀改成 .zip 格式解压后（也可以直接解压，apk 文件本质是个 zip 文件），里面就有 class.dex 文件，由于 Android 的 65K 问题（不要纠结是 64K 还是 65K），使用 MultiDex 就会生成多个 dex 文件
- 当 Android 系统安装一个应用的时候，会针对不同平台对 Dex 进行优化，这个过程由一个专门的工具来处理，叫 DexOpt 。DexOpt 是在第一次加载 Dex 文件的时候执行的，该过程会生成一个 ODEX 文件，即 Optimised Dex。执行 ODEX 的效率会比直接执行 Dex 文件的效率要高很多，加快 App 的启动和响应
- 最终在Android虚拟机上执行的并非Java 字节码，而是另一种字节码：**dex 格式的字节码**。在编译 Java 代码之 后 ，通过 Android 平台上的工具可以将 Java 字节码转换成 Dex 字节码。

总之，Android 中的 Dalvik/ART **无法像 JVM 那样 直接 加载 class 文件和 jar 文件中的 class**，需要通过 dx 工具来优化转换成 Dalvik byte code 才行，只能通过 **dex 或者 包含 dex 的jar、apk 文件来加载**（注意 odex 文件后缀可能是 .dex 或 .odex，也属于 dex 文件），因此 Android 中的 ClassLoader 工作就**主要交给了 BaseDexClassLoader** 来处理

> 注：如果 jar 文件包含有 dex 文件，此时 jar 文件也是可以用来加载的，不过实际加载的还是其中的 dex 文件，不要弄混淆了。 注：有的Android应用能直接加载.jar文件，那是因为这个.jar文件已经经过优化，只不过后缀名没改（其实已经是.dex文件）

更多Odex请参看[ODEX格式及生成过程](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F242abfb7eb7f)、[ART和Dalvik](https://link.juejin.cn?target=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F9593527%2Fwhat-are-odex-files-in-android)、 [what-are-odex-files-in-android](https://link.juejin.cn?target=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F9593527%2Fwhat-are-odex-files-in-android)        

## 二、ClassLoader的类型

Android中的ClassLoader类型和Java中的ClassLoader类型类似，也分为两种类型:

- 系统内置 
  - `BootClassLoader`
  - `PathClassLoader`
  - `DexClassLoader`
- 用户自定义 
  - `...`

运行一个Android程序需要用到几种类型的类加载器呢？如下所示

```
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ClassLoader loader = MainActivity.class.getClassLoader();
        while (loader != null) {
            Log.d("liuwangshu",loader.toString());//1
            loader = loader.getParent();
        }
    }
}复制代码
```

可以看到有两种类加载器：

1. 一种是

   ```
   PathClassLoader
   ```

   . 在 Android 中，App 安装到手机后，apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的 

   - DexPathList中包含了很多apk的路径，其中/data/app/com.example.liuwangshu.moonclassloader-2/base.apk就是示例应用安装在手机上的位置。关于DexPathList后续文章会进行介绍

2. 另一种则是`BootClassLoader`。 

```
10-07 07:23:02.835 8272-8272/? D/liuwangshu: dalvik.system.PathClassLoader[DexPathList
        [
            [zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/base.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_dependencies_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_0_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_1_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_2_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_3_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_4_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_5_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_6_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_7_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_8_apk.apk”,
            zipfile“/data/app/com.example.liuwangshu.moonclassloader-2/split_lib_slice_9_apk.apk”],

            nativeLibraryDirectories=[/data/app/com.example.liuwangshu.moonclassloader-2/lib/x86,
            /vendor/lib,
            /system/lib]
        ]
    ]
10-07 07:23:02.835 8272-8272/? D/liuwangshu: java.lang.BootClassLoader@e175998
复制代码
```

## 三、ClassLoader的继承关系

和Java中的ClassLoader一样，虽然系统所提供的类加载器有3种类型，但是系统提供的ClassLoader相关类却不只3个。ClassLoader的继承关系如下图所示

![这里写图片描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210807215404.awebp) 
 【processon 图Android ClassLoader的继承关系 (2)】

可以看到上面一共有8个ClassLoader相关类，其中有一些和Java中的ClassLoader相关类十分类似，下面简单对它们进行介绍：

- `ClassLoader`是一个抽象类，其中定义了ClassLoader的主要功能。

- `BootClassLoader`是它的内部类，用于**预加载preload()**常用类，加载一些系统Framework层级需要的类，我们的Android应用里也需要用到一些系统的类等

- ```
  SecureClassLoader
  ```

  类和JDK8中的SecureClassLoader类的代码是一样的，它继承了抽象类ClassLoader。SecureClassLoader并不是ClassLoader的实现类，而是拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。 

  - `URLClassLoader`类和JDK8中的URLClassLoader类的代码是一样的，它继承自SecureClassLoader，用来通过URl路径从jar文件和文件夹中加载类和资源。**在Android中基本无法使用**

- ```
  BaseDexClassLoader
  ```

  继承自ClassLoader，是抽象类ClassLoader的具体实现类，PathClassLoader和DexClassLoader都继承它。 

  - `PathClassLoader`加载系统类和应用程序的类，如果是加载非系统应用程序类，则会加载data/app/目录下的dex文件以及包含dex的apk文件或jar文件
  - `DexClassLoader` 可以加载自定义的dex文件以及包含dex的apk文件或jar文件，也支持从SD卡进行加载
  - `InMemoryDexClassLoader`是Android8.0新增的类加载器，继承自BaseDexClassLoader，用于加载内存中的dex文件。

### 3.1 ClassLoader

`java.lang.ClassLoader`是所有ClassLoader的最终父类.

#### 3.1.1 构造方法

- 构造方法主要以下两种 
  1. 显式传入一个 `父构造器实例`
  2. 无参默认构造法 
     - 还是调用了 `需要传入父构造器实例` 的构造方法
     - 不像java虚拟机里父构造器为空时默认的父构造器为`Bootstrap ClassLoader`，Android中默认无父构造器传入的情况下，默认父构造器为一个`PathClassLoader`且此PathClassLoader父构造器为`BootClassLoader`。

```
package java.lang;

public abstract class ClassLoader {

    private ClassLoader parent;

    //【1】
    protected ClassLoader(ClassLoader parentLoader) {
        this(parentLoader, false);
    }
    ClassLoader(ClassLoader parentLoader, boolean nullAllowed) {
        if (parentLoader == null && !nullAllowed) {
            throw new NullPointerException("parentLoader == null && !nullAllowed");
        }
        parent = parentLoader;
    }

    /**【2】
     * Constructs a new instance of this class with the system class loader as
     * its parent.
     */
    protected ClassLoader() {
        this(getSystemClassLoader(), false);
    }

    public static ClassLoader getSystemClassLoader() {
        return SystemClassLoader.loader;
    }

    static private class SystemClassLoader {
        public static ClassLoader loader = ClassLoader.createSystemClassLoader();
    }

    /**
     * Create the system class loader. Note this is NOT the bootstrap class
     * loader (which is managed by the VM). We use a null value for the parent
     * to indicate that the bootstrap loader is our parent.
     */
    private static ClassLoader createSystemClassLoader() {
        String classPath = System.getProperty("java.class.path", ".");

        // TODO Make this a java.net.URLClassLoader once we have those?
        return new PathClassLoader(classPath, BootClassLoader.getInstance());
    }
}复制代码
```

#### 3.1.2 loadclass与双亲委托

- ClassLoader中重要的方法是

  ```
  loadClass(String name)
  ```

  ，其他的子类都继承了此方法且没有进行复写 

  - 和java虚拟机中常见的双亲委派模型一致的
  - 在加载类时首先判断这个类是否之前被加载过，如果有则直接返回，如果没有则首先尝试让`parent ClassLoader`进行加载,加载不成功才在自己的`findClass()`中进行加载
  - 这种模型并不是一个强制性的约束模型，比如你可以继承ClassLoader复写loadCalss方法来破坏这种模型，只不过双亲委派模是一种**被推荐的实现类加载器的方式**，而且jdk1.2以后已经不提倡用户在覆盖loadClass方法，而应该把自己的类加载逻辑写到findClass中

```
public Class<?> loadClass(String className) throws ClassNotFoundException {
        return loadClass(className, false);
    }

protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        //【1】是否已经加载过
        Class<?> clazz = findLoadedClass(className);

        if (clazz == null) {
            ClassNotFoundException suppressed = null;
            try {
                //【2】向上委托
                clazz = parent.loadClass(className, false);
            } catch (ClassNotFoundException e) {
                suppressed = e;
            }

            if (clazz == null) {
                try {
                    //【3】向下查找
                    clazz = findClass(className);
                } catch (ClassNotFoundException e) {
                    e.addSuppressed(suppressed);
                    throw e;
                }
            }
        }

        return clazz;
    }复制代码
```

- ```
  loadClass()
  ```

   和java的加载过程类似，但是具体实现变得不同，后文会讲到： 

  - findLoadedClass()
  - findClass()
  - defineClass() 
     不同于java，`Android ClassLoader#defineClass(String name)`该方法被废弃使用，改为使用`DexPathList#findClass(String name)`

```
//【1】
protected final Class<?> findLoadedClass(String className) {
        ClassLoader loader;
        if (this == BootClassLoader.getInstance())
            loader = null;
        else
            loader = this;
        return VMClassLoader.findLoadedClass(loader, className);
    }   

//【2】不同子类复写此方法，实现不同路径的查找    
protected Class<?> findClass(String className) throws ClassNotFoundException {
    throw new ClassNotFoundException(className);
}

//【3】将class二进制内容转换成Class对象，如果不符合要求的会抛出各种异常
@Deprecated
protected final Class<?> defineClass(byte[] classRep, int offset, int length)
        throws ClassFormatError {
    throw new UnsupportedOperationException("can't load this type of class file");
}复制代码
```

#### 3.1.3 其他Utils方法

ClassLoader还提供了一系列的Utils方法

- 包缓存 
  - `Package definePackage()`,根据参数构建一个`java.lang.Package`实例
  - `Package[] getPackages()`： 返回当前ClassLoader知道的所有包
  - `Package getPackage(String name)`： 返回指定当前ClassLoader知道的指定路径的包

```
/**
 * The packages known to the class loader.
 */
private Map<String, Package> packages = new HashMap<String, Package>();  

//Returns the package with the specified name. Package information is searched in this class loader.
protected Package getPackage(String name) {
    synchronized (packages) {
        return packages.get(name);
    }
}
//Returns all the packages known to this class loader.
protected Package[] getPackages() {
    synchronized (packages) {
        Collection<Package> col = packages.values();
        Package[] result = new Package[col.size()];
        col.toArray(result);
        return result;
    }
}   

/**
* Defines and returns a new {@code Package} using the specified information. 
* If {@code sealBase} is {@code null}, the package is left unsealed. Otherwise, the package is sealed using this URL.
*/
protected Package definePackage(String name, String specTitle, String specVersion,
        String specVendor, String implTitle, String implVersion, String implVendor, URL sealBase)
        throws IllegalArgumentException {

    synchronized (packages) {
        if (packages.containsKey(name)) {
            throw new IllegalArgumentException("Package " + name + " already defined");
        }

        Package newPackage = new Package(name, specTitle, specVersion, specVendor, implTitle,
                implVersion, implVendor, sealBase);

        packages.put(name, newPackage);

        return newPackage;
    }
}复制代码
```

- Resource的双亲委托模型 
  - `URL getResource(String resName)`**查找具备resName的资源的全路径，具体怎么查找由子类复写该方法**
  - `URL findResource(String resName)`：
  - `Enumeration<URL> getResources(String resName)`
  - `Enumeration<URL> findResources(String resName)`
  - InputStream getResourceAsStream(String resName)

```
public URL getResource(String resName) {
    URL resource = parent.getResource(resName);
    if (resource == null) {
        resource = findResource(resName);
    }
    return resource;
}

protected URL findResource(String resName) {
    return null;
}
复制代码
```

- ```
  String findLibrary(String libName)
  ```

  ：返回一个全路径，指向

  ```
  java.library.path
  ```

  目录下的

  ```
  libName
  ```

  名称的library 

  - 具体实现由子类复写

```
protected String findLibrary(String libName) {
    return null;
}复制代码
```

### 3.2 BootClassLoader

Android系统启动时会使用BootClassLoader来预加载常用类.

与Java中的BootClassLoader不同，**它并不是由C/C++代码实现，而是由Java实现的**，BootClassLoade的代码如下所示

- BootClassLoader是ClassLoader的内部类，并继承自ClassLoader。
- BootClassLoader是一个单例类
- 需要注意的是BootClassLoader的访问修饰符是默认的，只有在同一个包中才可以访问，因此我们在应用程序中是无法直接调用的



```
libcore/ojluni/src/main/java/java/lang/ClassLoader.java
```



```
class BootClassLoader extends ClassLoader {

    private static BootClassLoader instance;

    @FindBugsSuppressWarnings("DP_CREATE_CLASSLOADER_INSIDE_DO_PRIVILEGED")
    public static synchronized BootClassLoader getInstance() {
        if (instance == null) {
            instance = new BootClassLoader();
        }
        return instance;
    }

    public BootClassLoader() {
        super(null, true);
    }
    ...
}复制代码
```

我们来看下核心的loadClass的实现

```
@Override
protected Class<?> loadClass(String className, boolean resolve)
       throws ClassNotFoundException {
    Class<?> clazz = findLoadedClass(className);//父类的方法，上边已经讲过

    if (clazz == null) {
        clazz = findClass(className);
    }

    return clazz;
}

@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    return Class.classForName(name, false, null);
}

//# 最终调用 java.lang.Class中的native方法
static native Class<?> classForName(String className, boolean shouldInitialize,
            ClassLoader classLoader) throws ClassNotFoundException;
复制代码
```

### 3.3 SecureClassLoader

SecureClassLoader类和JDK8中的SecureClassLoader类的代码是一样的，它继承了抽象类ClassLoader。

SecureClassLoader并不是ClassLoader的实现类，而是拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。

- URLClassLoader类和JDK8中的URLClassLoader类的代码是一样的，它继承自SecureClassLoader，用来通过URl路径从jar文件和文件夹中加载类和资源。**但是由于 dalvik 不能直接识别jar，所以在 Android 中无法使用这个加载器**
- JSClassLoader
- AppClassLoader

### 3.4 BaseDexClassLoader

> 在这里为了方便理解，我们将dex文件以及包含dex的apk文件或jar文件统称为dex相关文件

PathClassLoader和DexClassLoader都继承自BaseDexClassLoader,其中的主要逻辑都是在BaseDexClassLoader完成的。这些源码在[java/dalvik/system](https://link.juejin.cn?target=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F8.0.0_r4%2Fxref%2Flibcore%2Fdalvik%2Fsrc%2Fmain%2Fjava%2Fdalvik%2Fsystem%2FBaseDexClassLoader.java)中

- 构造函数。BaseDexClassLoader的构造函数包含四个参数，分别为： 

  - `dexPath`，指目标类所在的APK或jar文件的路径 
     类装载器将从该路径中寻找指定的目标类,该类必须是APK或jar的全路径. 
     如果要包含多个路径,路径之间必须使用特定的分割符分隔,特定的分割符可以使用System.getProperty(“path.separtor”)获得。 
     上面”支持加载APK、DEX和JAR，也可以从SD卡进行加载”指的就是这个路径，最终做的是将dexPath路径上的文件ODEX优化到内部位置optimizedDirectory，然后，再进行加载的。

  - ```
    File optimizedDirectory
    ```

    - optimizedDirectory是用来缓存我们需要加载的dex文件的，并创建一个DexFile对象，如果它为null，那么会直接使用dex文件原有的路径来创建DexFile对象
    - 由于dex文件被包含在APK或者Jar文件中,因此在装载目标类之前需要先从APK或Jar文件中解压出dex文件,该参数就是制定解压出的dex 文件存放的路径。
    - 这也是对apk中dex根据平台进行ODEX优化的过程。其实APK是一个程序压缩包，里面包含dex文件，ODEX优化就是把包里面的执行程序提取出来，就变成ODEX文件，因为你提取出来了，系统第一次启动的时候就不用去解压程序压缩包的程序，少了一个解压的过程。这样的话系统启动就加快了。
    - 为什么说是第一次呢？是因为DEX版本的也只有第一次会解压执行程序到 /data/dalvik-cache（针对PathClassLoader）或者optimizedDirectory(针对DexClassLoader）目录，之后也是直接读取目录下的的dex文件，所以第二次启动就和正常的差不多了。当然这只是简单的理解，实际生成的ODEX还有一定的优化作用。
    - **无论哪种动态加载，ClassLoader只能加载内部存储路径中的dex文件，所以这个路径必须为内部路径。**

  - `libPath`,指目标类中所使用的C/C++库存放的路径

  - `classload`,是指该装载器的父装载器 
     一般为当前执行类的装载器，例如在Android中以context.getClassLoader()作为父装载器。

> DexClassLoader可以指定自己的optimizedDirectory，所以它可以加载外部的dex，因为这个dex会被复制到内部路径的optimizedDirectory；而PathClassLoader没有optimizedDirectory，所以它只能加载内部的dex，这些大都是存在系统中已经安装过的apk里面的 

```
package dalvik.system;
public class BaseDexClassLoader extends ClassLoader {

    private final DexPathList pathList;

    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
        String librarySearchPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null);

        if (reporter != null) {
            reporter.report(this.pathList.getDexPaths());
        }
    }

    public BaseDexClassLoader(ByteBuffer[] dexFiles, ClassLoader parent) {
        // TODO We should support giving this a library search path maybe.
        super(parent);
        this.pathList = new DexPathList(this, dexFiles);
    }

    @Override public String toString() {
        return getClass().getName() + "[" + pathList + "]";
    }
}复制代码
```

- BaseDexClassLoader的相关操作都是委托DexPathList执行的

```
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException(
                    "Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }

    @Override
    protected URL findResource(String name) {
        return pathList.findResource(name);
    }

    @Override
    protected Enumeration<URL> findResources(String name) {
        return pathList.findResources(name);
    }

    @Override
    public String findLibrary(String name) {
        return pathList.findLibrary(name);
    }复制代码
```

#### 3.4.1 ClassLoader加载class的过程

BaseDexClassLoader中有个pathList对象，pathList中包含一个`DexFile的数组dexElements`

- dexElements数组就是odex文件的集合 
  - `odex文件`是 dexPath指向的原始dex(.apk,.zip,.jar等)文件在optimizedDirectory文件夹中生成相应的优化后的文件
  - 如果不分包一般这个数组只有一个Element元素，也就只有一个DexFile文件

对于类加载呢，就是遍历这个集合，通过DexFile去寻找，并最终调用native方法的defineClass

```
#BaseDexClassLoader  
@Override  
protected Class<?> findClass(String name) throws ClassNotFoundException {   
    Class clazz = pathList.findClass(name);  
    if (clazz == null) {   
        throw new ClassNotFoundException(name);   
    }   
    return clazz;  
}  

#DexPathList  
public Class findClass(String name) {   
    for (Element element : dexElements) {   
        DexFile dex = element.dexFile;  
        if (dex != null) {   
            Class clazz = dex.loadClassBinaryName(name, definingContext);   
          if (clazz != null) {   
              return clazz;   
          }   
        }   
    }   
    return null;  
}  

#DexFile  
public Class loadClassBinaryName(String name, ClassLoader loader) {   
    return defineClass(name, loader, mCookie);  
}  
private native static Class defineClass(String name, ClassLoader loader, int cookie);  复制代码
```

#### 3.4.2 DexPathList

##### 构造过程

- DexPathList 的构造方法
  - String dexPath：之前传进来的包含 dex 的 apk、jar、dex 的路径集
  - String libraryPath:native 库的路径集
  - File optimizedDirectory: 缓存优化的 odex 文件的路径

```
final class DexPathList {  

    private static final String DEX_SUFFIX = ".dex";
    private static final String zipSeparator = "!/";

    private final ClassLoader definingContext;
    private Element[] dexElements;
    private final NativeLibraryElement[] nativeLibraryPathElements;

    public DexPathList(ClassLoader definingContext, String dexPath, String libraryPath, File optimizedDirectory) {

        ...
        this.definingContext = definingContext;
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext);


    }

}复制代码
```

- 调用 

  ```
  makePathElements()
  ```

   方法生成一个Element[] dexElements 数组 

  - Element 是 DexPathList 的一个嵌套类

```
static class Element {
    private final File dir;
    private final boolean isDirectory;
    private final File zip;
    private final DexFile dexFile;
    private ZipFile zipFile;
    private boolean initialized;
}

rivate static Element[] makePathElements(List<File> files, File optimizedDirectory,
                                          List<IOException> suppressedExceptions) {
    List<Element> elements = new ArrayList<>();
    // 遍历所有的包含 dex 的文件
    for (File file : files) {
        File zip = null;
        File dir = new File("");
        DexFile dex = null;
        String path = file.getPath();
        String name = file.getName();
        // 判断是不是 zip 类型
        if (path.contains(zipSeparator)) {
            String split[] = path.split(zipSeparator, 2);
            zip = new File(split[0]);
            dir = new File(split[1]);
        } else if (file.isDirectory()) {
            // 如果是文件夹,则直接添加 Element,这个一般是用来处理 native 库和资源文件
            elements.add(new Element(file, true, null, null));
        } else if (file.isFile()) {
            // 直接是 .dex 文件,而不是 zip/jar 文件(apk 归为 zip),则直接加载 dex 文件
            if (name.endsWith(DEX_SUFFIX)) {
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ex) {
                    System.logE("Unable to load dex file: " + file, ex);
                }
            } else {
                // 如果是 zip/jar 文件(apk 归为 zip),则将 file 值赋给 zip 字段,再加载 dex 文件
                zip = file;
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException suppressed) {
                    suppressedExceptions.add(suppressed);
                }
            }
        } else {
            System.logW("ClassLoader referenced unknown path: " + file);
        }
        if ((zip != null) || (dex != null)) {
            elements.add(new Element(dir, false, zip, dex));
        }
    }
    // list 转为数组
    return elements.toArray(new Element[elements.size()]);
}复制代码
```

- loadDexFile() 方法最终会调用 JNI 层的方法来读取 dex 文件，这里不再深入探究，有兴趣的可以阅读 [从源码分析 Android dexClassLoader 加载机制原理](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fnanzhiwen666%2Farticle%2Fdetails%2F50515895) 这篇文章深入了解

```
private static DexFile loadDexFile(File file, File optimizedDirectory) throws IOException {
    if (optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
    }
} 

/**
 * Converts a dex/jar file path and an output directory to an      
 * output file path for an associated optimized dex file.
 */
private static String optimizedPathFor(File path, File optimizedDirectory) {
    String fileName = path.getName();
    if (!fileName.endsWith(DEX_SUFFIX)) {
        int lastDot = fileName.lastIndexOf(".");
        if (lastDot < 0) {
            fileName += DEX_SUFFIX;
        } else {
            StringBuilder sb = new StringBuilder(lastDot + 4);
            sb.append(fileName, 0, lastDot);
            sb.append(DEX_SUFFIX);
            fileName = sb.toString();
        }
    }
    File result = new File(optimizedDirectory, fileName);
    return result.getPath();
}复制代码
```

##### 加载类的过程

- 【核心】DexPathList 的 findClass() 方法，其根据传入的完整的类名来加载对应的 class

```
public Class findClass(String name, List<Throwable> suppressed) {
    // 遍历 dexElements 数组，依次寻找对应的 class，一旦找到就终止遍历
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    // 抛出异常
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
} 复制代码
```

> 这里有关于热修复实现的一个点，就是将补丁 dex 文件放到 dexElements 数组前面，这样在加载 class 时，优先找到补丁包中的 dex 文件，加载到 class 之后就不再寻找，从而原来的 apk 文件中同名的类就不会再使用，从而达到修复的目的，虽然说起来较为简单，但是实现起来还有很多细节需要注意

- loadClassBinaryName中调用了Native方法

  ```
  defineClass()
  ```

  加载类 

  - 标准JVM中，ClassLoader是用defineClass加载类的，而Android中defineClass被弃用了，改用了loadClass方法，而且加载类的过程也挪到了DexFile中，在DexFile中加载类的具体方法也叫defineClass，相信这也是维护代码可读性

```
public Class loadClassBinaryName(String name, ClassLoader loader) {
        return defineClass(name, loader, mCookie);
}

private native static Class defineClass(String name, ClassLoader loader, int cookie);复制代码
```

至此，BaseDexClassLader 寻找 class 的路线就清晰了：

1. 当传入一个完整的类名，调用 BaseDexClassLader 的 findClass(String name) 方法

2. BaseDexClassLader 的 findClass 方法会交给 DexPathList 的 `findClass(String name, List<Throwable> suppressed` 方法处理

3. 在 DexPathList 方法的内部，会遍历 dexFile ，通过 DexFile 的 

   ```
   dex.loadClassBinaryName(name, definingContext, suppressed)
   ```

    来完成类的加载

#### 3.4.3 DexClassLoader

DexClassLoader可以加载dex文件以及包含dex的apk文件或jar文件，也支持从SD卡进行加载，这也就意味着DexClassLoader可以在应用未安装的情况下加载dex相关文件。因此，它是热修复和插件化技术的基础

> 上面说dalvik不能直接识别jar,DexClassLoader却可以加载jar文件,这难道不矛盾吗?其实在BaseDexClassLoader里对”.jar”,”.zip”,”.apk”,”.dex”后缀的文件最后都会生成一个对应的dex文件,所以最终处理的还是dex文件,而URLClassLoader并没有做类似的处理

- DexClassLoader 也继承自BaseDexClassLoader ，方法实现也都在BaseDexClassLoader中

libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java

```
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
        }
}复制代码
```

我们再次强调下 `String librarySearchPath` 的意义：

我们知道应用程序第一次被加载的时候，为了提高以后的启动速度和执行效率，Android系统会对dex相关文件做一定程度的优化，并生成一个ODEX文件，此后再运行这个应用程序的时候，只要加载优化过的ODEX文件就行了，省去了每次都要优化的时间，而参数optimizedDirectory就是代表存储ODEX文件的路径，这个路径必须是一个内部存储路径。 

#### 3.4.4 PathClassLoader

Android系统使用PathClassLoader来加载系统类和应用程序的类，如果是加载非系统应用程序类，则会加载data/app/目录下的dex文件以及包含dex的apk文件或jar文件，不管是加载那种文件，最终都是要加载dex文件

也就是说，在 Android 中，App 安装到手机后，apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的

- PathClassLoader不建议开发直接使用
- PathClassLoader继承自BaseDexClassLoader，很明显PathClassLoader的方法实现都在BaseDexClassLoader中。
- 从PathClassLoader的构造方法也可以看出它遵循了双亲委托模式



```
libcore/dalvik/src/main/java/dalvik/system/PathClassLoader.java
```



```
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}复制代码
```

PathClassLoader的构造方法有三个参数：

1. dexPath：dex文件以及包含dex的apk文件或jar文件的路径集合，多个路径用文件分隔符分隔，默认文件分隔符为‘：’。
2. librarySearchPath：包含 C/C++ 库的路径集合，多个路径用文件分隔符分隔分割，可以为null。
3. parent：ClassLoader的parent [双亲委托模式]

PathClassLoader没有参数optimizedDirectory，这是因为PathClassLoader已经默认了参数optimizedDirectory的路径为：`/data/dalvik-cache`

> 很多博客里说PathClassLoader只能加载已安装的apk的dex，其实这说的应该是在dalvik虚拟机上，在art虚拟机上PathClassLoader可以加载未安装的apk的dex（在art平台上已验证），然而在/data/dalvik-cache 确未找到相应的dex文件，怀疑是art虚拟机判断apk未安装，所以只是将apk优化后的odex放在内存中，之后进行释放，这只是个猜想，希望有知道的可以告知一下。因为dalvik上无法使用，所以我们也没法使用

## 四、ClassLoader的实例化顺序

```
BootClassLoader`是在Zygote进程的入口方法中创建的，`PathClassLoader`则是在Zygote进程创建SystemServer进程时创建的，查找路径为`java.library.path
```

### 4.1 BootClassLoader的创建

BootClassLoader是在何时被创建的呢？这得先从Zygote进程开始说起，不了解Zygote进程的可以查看[（4.1.52）Android启动流程分析](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Ffei20121106%2Farticle%2Fdetails%2F78712943)

ZygoteInit的main方法如下所示



```
frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
```



```
 public static void main(String argv[]) {
   ...
        try {
             ...
                preload(bootTimingsTraceLog);
             ... 
        }
    }

复制代码
```

main方法是ZygoteInit入口方法，其中调用了ZygoteInit的`preload`方法，preload方法中又调用了ZygoteInit的`preloadClasses`方法，如下所示。



```
frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
```



```
private static void preloadClasses() {
        final VMRuntime runtime = VMRuntime.getRuntime();
        InputStream is;
        try {
            is = new FileInputStream(PRELOADED_CLASSES);//【1】
        } catch (FileNotFoundException e) {
            Log.e(TAG, "Couldn't find " + PRELOADED_CLASSES + ".");
            return;
        }
        ...
        try {
            BufferedReader br
                = new BufferedReader(new InputStreamReader(is), 256);//【2】

            int count = 0;
            String line;
            while ((line = br.readLine()) != null) {//【3】
                line = line.trim();
                if (line.startsWith("#") || line.equals("")) {
                    continue;
                }
                  Trace.traceBegin(Trace.TRACE_TAG_DALVIK, line);
                try {
                    if (false) {
                        Log.v(TAG, "Preloading " + line + "...");
                    }
                    Class.forName(line, true, null);//【4】
                    count++;
                } catch (ClassNotFoundException e) {
                    Log.w(TAG, "Class not found for preloading: " + line);
                } 
        ...
        } catch (IOException e) {
            Log.e(TAG, "Error reading " + PRELOADED_CLASSES + ".", e);
        } finally {
            ...
        }
    }
```

preloadClasses方法用于Zygote进程初始化时预加载常用类

1. 【`注释1处`】将`/system/etc/preloaded-classes`文件封装成FileInputStream，preloaded-classes文件中存有预加载类的目录，这个文件在系统源码中的路径为`frameworks/base/preloaded-classes`.

这里列举一些preloaded-classes文件中的预加载类名称，如下所示

```
android.app.ApplicationLoaders
android.app.ApplicationPackageManager
android.app.ApplicationPackageManager$OnPermissionsChangeListenerDelegate
android.app.ApplicationPackageManager$ResourceName
android.app.ContentProviderHolder
android.app.ContentProviderHolder$1
android.app.ContextImpl
android.app.ContextImpl$ApplicationContentResolver
android.app.DexLoadReporter
android.app.Dialog
android.app.Dialog$ListenersHandler
android.app.DownloadManager
android.app.Fragment
```

可以看到preloaded-classes文件中的预加载类的名称有很多都是我们非常熟知的。预加载属于拿空间换时间的策略，Zygote环境配置的越健全越通用，应用程序进程需要单独做的事情也就越少，预加载除了预加载类，还有预加载资源和预加载共享库，因为不是本文重点，这里就不在延伸讲下去了。

1. 【`注释2处`】，将FileInputStream封装为BufferedReade
2. 【`注释3处`】遍历BufferedReader，读出所有预加载类的名称，每读出一个预加载类的名称就调用【`注释4处`】的代码加载该类，Class的forName方法如下所示

```
libcore/ojluni/src/main/java/java/lang/Class.java
 @CallerSensitive
 public static Class<?> forName(String name, boolean initialize,
                                   ClassLoader loader)
        throws ClassNotFoundException
    {
        if (loader == null) {
            loader = BootClassLoader.getInstance();//【1】创建了BootClassLoader，并将BootClassLoader实例传入到了注释2处的classForName方法中
        }
        Class<?> result;
        try {
            result = classForName(name, initialize, loader);//【2】 classForName方法是Native方法，它的实现由c/c++代码来完成
        } catch (ClassNotFoundException e) {
            Throwable cause = e.getCause();
            if (cause instanceof LinkageError) {
                throw (LinkageError) cause;
            }
            throw e;
        }
        return result;
    }

 @FastNative
 static native Class<?> classForName(String className, boolean shouldInitialize,
            ClassLoader classLoader) throws ClassNotFoundException;复制代码
```

### 4.2 PathClassLoader的创建
PathClassLoader的创建也得从Zygote进程开始说起，Zygote进程启动SyetemServer进程时会调用ZygoteInit的`startSystemServer()`方法，如下所示。



```
frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
```



```
private static boolean startSystemServer(String abiList, String socketName)
           throws MethodAndArgsCaller, RuntimeException {
    ...
        int pid;
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);//2
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
            /*1*/
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }
       if (pid == 0) {//【2】
           if (hasSecondZygote(abiList)) {
               waitForSecondaryZygote(socketName);
           }
           handleSystemServerProcess(parsedArgs);//【3】
       }
       return true;
   }复制代码
```

`注释1`处，Zygote进程通过`forkSystemServer()`方法fork自身创建子进程（SystemServer进程）。`注释2处`如果forkSystemServer方法返回的pid等于0，说明当前代码是在新创建的SystemServer进程中执行的，接着就会执行`注释3处`的`handleSystemServerProcess()`方法： 



```
frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
```



```
 private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws Zygote.MethodAndArgsCaller {

    ...
        if (parsedArgs.invokeWith != null) {
           ...
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);//【4】
                Thread.currentThread().setContextClassLoader(cl);
            }
            ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }
    }复制代码
```

`注释4处`调用了`createPathClassLoader()`方法，如下所示

```
//frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
 static PathClassLoader createPathClassLoader(String classPath, int targetSdkVersion) {
      String libraryPath = System.getProperty("java.library.path");
      return PathClassLoaderFactory.createClassLoader(classPath,
                                                      libraryPath,
                                                      libraryPath,
                                                      ClassLoader.getSystemClassLoader(),
                                                      targetSdkVersion,
                                                      true /* isNamespaceShared */);
    }

//createPathClassLoader方法中又会调用PathClassLoaderFactory的createClassLoader方法，看来PathClassLoader是用工厂来进行创建的
//frameworks/base/core/java/com/android/internal/os/PathClassLoaderFactory.java
 public static PathClassLoader createClassLoader(String dexPath,
                                                    String librarySearchPath,
                                                    String libraryPermittedPath,
                                                    ClassLoader parent,
                                                    int targetSdkVersion,
                                                    boolean isNamespaceShared) {
        PathClassLoader pathClassloader = new PathClassLoader(dexPath, librarySearchPath, parent);
      ...
        return pathClassloader;
    }

复制代码
```

在PathClassLoaderFactory的createClassLoader方法中会创建PathClassLoader

另外，在 ApplicationLoaders 中用来加载系统安装过的 apk，用来加载 apk 内的 class ，其调用是在 `LoadApk` 类中的 `getClassLoader()` 方法中调用的，得到的也是 PathClassLoader

```
mClassLoader = ApplicationLoaders.getDefault().getClassLoader(zip, lib,
        mBaseClassLoader);复制代码
```

## 五、自定义ClassLoader

### 5.1 示例之 SD卡加载

从 SD 卡中动态加载一个包含 class.dex 的 jar 文件，加载其中的类，并调用其方法

- 新建一个 Java 项目，包含两个文件：`ISayHello.java` 和 `HelloAndroid.java`.

```
package com.jaeger;
public interface ISayHello {
   String say();
}

package com.jaeger;
public class HelloAndroid implements ISayHello {
   @Override
   public String say() {
       return "Hello Android";
   }
}复制代码
```

- 导出 jar 包
- 使用 SDK 目录 > platform-tools 里面的 dx 工具生成包含 class.dex 的 jar 包 
  - 将上一步生成的 `sayhello.jar` 放到 你的 SDK 下的 platform-tools 文件夹下，使用下面的命令生成 dex 化的 jar 文件，其中是 output 后面的`sayhello_dex.jar` 就是最终生成的 jar 包
  - `dx --dex --output=sayhello_dex.jar sayhello.jar`
- 将 sayhello_dex.jar 文件拷贝到手机存储空间的根目录，不一定是内存卡
- 新建一个 Android 项目，在 MainActivity 中添加如下的代码： 
  - DexClassLoader并不能直接加载外部存储的.dex文件，而是要先拷贝到内部存储里。这里的`dexPath`就是`.dex的外部存储路径`，而`optimizedDirectory`则是`内部路径`，libraryPath用null即可，parent则是要传入当前应用的ClassLoader，这与ClassLoader的“双亲代理模式”有关

```
package com.jaeger;
public interface ISayHello {
    String say();
}

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "TestClassLoader";
    private TextView mTvInfo;
    private Button mBtnLoad;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTvInfo = (TextView) findViewById(R.id.tv_info);
        mBtnLoad = (Button) findViewById(R.id.btn_load);
        mBtnLoad.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                // 获取到包含 class.dex 的 jar 包文件
                final File jarFile =
                    new File(Environment.getExternalStorageDirectory().getPath() + File.separator + "sayhello_dex.jar");

                // 如果没有读权限,确定你在 AndroidManifest 中是否声明了读写权限
                Log.d(TAG, jarFile.canRead() + "");

                if (!jarFile.exists()) {
                    Log.e(TAG, "sayhello_dex.jar not exists");
                    return;
                }

                //]【核心！！！！】jarFile.getAbsolutePath()
                // getCodeCacheDir() 方法在 API 21 才能使用,实际测试替换成 getExternalCacheDir() 等也是可以的
                // 只要有读写权限的路径均可
                DexClassLoader dexClassLoader =
                    new DexClassLoader(jarFile.getAbsolutePath(), getExternalCacheDir().getAbsolutePath(), null, getClassLoader());
                try {
                    // 加载 HelloAndroid 类
                    Class clazz = dexClassLoader.loadClass("com.jaeger.HelloAndroid");
                    // 强转成 ISayHello, 注意 ISayHello 的包名需要和 jar 包中的
                    ISayHello iSayHello = (ISayHello) clazz.newInstance();
                    mTvInfo.setText(iSayHello.say());
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                } catch (InstantiationException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}复制代码
```

这里需要注意几点：

- 因为需要从存储空间中读取 jar 文件，需要在 AndroidManifest 中声明读写权限
- ISayHello 接口的包名必须一致
- getCodeCacheDir() 方法在 API 21 才能使用,实际测试替换成 getExternalCacheDir() 等也是可以的
- 接下来就是运行，运行的结果如图，和预期的一样，完美收工。

示例代码以及 jar 包上传到 GitHub 了，请前往 [laobie/TestClassLoader](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Flaobie%2FTestClassLoader) 去查看

## 其他

### ART虚拟机的兼容性问题

Android Runtime（缩写为ART），在Android 5.0及后续Android版本中作为正式的运行时库取代了以往的Dalvik虚拟机。

ART能够把应用程序的字节码转换为机器码，是Android所使用的一种新的虚拟机。它与Dalvik的主要不同在于：

- Dalvik采用的是JIT技术，字节码都需要通过即时编译器（just in time ，JIT）转换为机器码，这会拖慢应用的运行效率
- ART采用Ahead-of-time（AOT）技术，应用在第一次安装的时候，字节码就会预先编译成机器码，这个过程叫做预编译。 
  - 即使用Android系统自带的dex2oat工具把APK里面的.dex文件转化成OAT文件，OAT文件是一种Android私有ELF文件格式，它不仅包含有从DEX文件翻译而来的本地机器指令，还包含有原来的DEX文件内容 
     ART同时也改善了性能、垃圾回收（Garbage Collection）、应用程序除错以及性能分析。但是请注意，运行时内存占用空间较少同样意味着编译二进制需要更高的存储

ART模式的系统里，同样存在DexClassLoader类，包名路径也没变，只不过它的具体实现与原来的有所不同，但是接口是一致的。实际上，ART运行时就是和Dalvik虚拟机一样，实现了一套完全兼容Java虚拟机的接口

### 动态加载类中的缓存干扰问题

如果你希望通过动态加载的方式，加载一个新版本的dex文件，使用里面的新类替换原有的旧类，从而修复原有类的BUG，那么你必须保证在加载新类的时候，旧类还没有被加载，因为如果已经加载过旧类，那么ClassLoader会一直优先使用旧类，因为会**先命中缓存**

如果旧类总是优先于新类被加载，我们也可以使用一个与加载旧类的ClassLoader没有树的继承关系的另一个ClassLoader来加载新类，因为ClassLoader只会检查其Parent有没有加载过当前要加载的类，如果两个ClassLoader没有继承关系，那么旧类和新类都能被加载

不过这样一来又有另一个问题了，在Java中，只有当两个实例的类名、包名以及加载其的ClassLoader都相同，才会被认为是同一种类型。上面分别加载的新类和旧类，虽然包名和类名都完全一样，但是由于加载的ClassLoader不同，所以并不是同一种类型，在实际使用中可能会出现类型不符异常

同一个Class = 相同的 ClassName + PackageName + ClassLoader

这个在采用动态加载功能的开发中容易出现，请注意









# 参考

https://blog.csdn.net/hxpjava1/article/details/55189730

https://juejin.cn/post/6844903621109252109#heading-23