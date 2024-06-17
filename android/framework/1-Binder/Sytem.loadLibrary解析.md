# 概述

在android我们通常使用System.loadLibrary或者System.load来加载so文件，比如

```java
    //加载的是libnative-lib.so，注意的是这边只需要传入"native-lib"
    System.loadLibrary("native-lib");
    //传入的是so文件完整的绝对路径
    System.load("/data/data/应用包名/lib/libnative-lib.so")

```

- loadLibrary传入的是编译脚本指定生成的so文件的名称,而load需要传入完整的so文件所在的绝对路径；
- load相比loadLibrary少了路径查找的过程；
- load并不是随便路径都可以，只支持应用本地存储路径/data/data/${package-name}/，或者是系统lib路径system/lib等,这2类路径；
- load如果传入的是sdcard路径，则会导致加载失败，可以采用将sdcard下的so文件复制到应用本地存储路径下进行加载；
- loadLibrary加载的都是一开始就已经打包进apk或系统的so文件了，而load可以是一开始就打包进来的so文件，也可以是后续从网络下载，外部导入的so文件。
- 最终都是调用nativeLoad加载指定路径的so文件；

这边详细分析源码来理解下，系统如何根据传入的文件名，来匹配到对应路径下的so文件

## 重点整理

1. loadLibrary传入的so文件名不需要包含开头的"lib"和结尾的".so"(如果包含，会导致找不到正确的so)，因为后续源码中在拼接成完整路径的时候会自动帮我们补上；
2. loadLibrary传入的不能是路径，否则会抛出异常UnsatisfiedLinkError("Directory separator should not appear in library name: " + libname)},该方法会自动从符合的路径下根据so文件名来查找;
3. loadLibrary查找so时，会优先从应用本地路径下(/data/data/${package-name}/lib/arm/)进行查找 不存在的话才会从系统lib路径下(/system/lib、/vendor/lib等)进行查找;
4. 重复调用loadLibrar,load并不会重复加载so，会优先从已加载的缓存中读取，所以只会加载一次；
5. 加载成功后会去搜索so是否有"JNI_OnLoad"，有的话则进行调用，所以"JNI_OnLoad"只会在加载成功后被主动回调一次，一般可以用来做一些初始化的操作，比如动态注册jni相关方法等;

# 源码分析

首先查看loadLibrary的源码，发现调用到了Runtime的loadLibrary0方法,而load则调用了Runtime的load0方法

**[System.java] java.lang.System:**

```java
 public static void loadLibrary(String libname) {
    Runtime.getRuntime().loadLibrary0(VMStack.getCallingClassLoader(), libname);
}

public static void load(String filename) {
    Runtime.getRuntime().load0(VMStack.getStackClass1(), filename);
}

```

**[Runtime.java] java.lang.Runtime:**

源码上load0可以看做是loadLibrary0的简化版本，最终都会调用到nativeLoad,所以不做分析

```java
    synchronized void load0(Class<?> fromClass, String filename) {
        //传入的如果不是路径的话则抛出异常
        if (!(new File(filename).isAbsolute())) {
            throw new UnsatisfiedLinkError(
                "Expecting an absolute path of the library: " + filename);
        }
        if (filename == null) {
            throw new NullPointerException("filename == null");
        }
        //调用nativeLoad加载指定路径的so文件
        String error = nativeLoad(filename, fromClass.getClassLoader());
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
    }
```

下边针对loadLibrary0进行源码分析，这边主要有3方面的处理： 1. classLoader存在时，通过classLoader.findLibrary(libraryName)来获取存放指定so文件的路径； 2. classLoader不存在时，则通过getLibPaths()接口来获取 3. 最终调用nativeLoad加载指定路径的so文件

```java
//这边的ClassLoader一般是PathClassLoader.java
synchronized void loadLibrary0(ClassLoader loader, String libname) {
        //传入的是so文件名，如果是路径，路径会抛出异常
        if (libname.indexOf((int)File.separatorChar) != -1) {
            throw new UnsatisfiedLinkError(
    "Directory separator should not appear in library name: " + libname);
        }
        
        //1.如果classloader不为空，则从classloader中获取so文件路径
        String libraryName = libname;
        if (loader != null) {
            String filename = loader.findLibrary(libraryName);
            if (filename == null) {
                // It's not necessarily true that the ClassLoader used
                // System.mapLibraryName, but the default setup does, and it's
                // misleading to say we didn't find "libMyLibrary.so" when we
                // actually searched for "liblibMyLibrary.so.so".
                throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                               System.mapLibraryName(libraryName) + "\"");
            }
            String error = nativeLoad(filename, loader);
            if (error != null) {
                throw new UnsatisfiedLinkError(error);
            }
            return;
        }
        
        //否则通过getLibPaths方式获取到so路径
        //mapLibraryName会将fileName会变成 "lib" + libraryName + ".so",这也是为什么我们system.loadLibrary不需要填写lib和so
        
        String filename = System.mapLibraryName(libraryName);
        List<String> candidates = new ArrayList<String>();
        String lastError = null;
        for (String directory : getLibPaths()) {
            //将文件夹路径和so文件名组成绝对路径
            String candidate = directory + filename;
            candidates.add(candidate);
            //判断这个文件是否有效
            if (IoUtils.canOpenReadOnly(candidate)) {
                //尝试加载这个so文件
                String error = nativeLoad(candidate, loader);
                if (error == null) {
                    return; // We successfully loaded the library. Job done.
                }
                lastError = error;
            }
        }
        
        //没找到指定so对应路径的话，抛出UnsatisfiedLinkError异常
        if (lastError != null) {
            throw new UnsatisfiedLinkError(lastError);
        }
        throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);
    }
复制代码
```

## 查找so文件所在绝对路径(ClassLoader存在时)

PathClassLoader 实际上调用的是父类BaseDexClassLoader的findLibrary接口

### [BaseDexClassLoader.java]

[BaseDexClassLoader.java]位于AOSP中platform/libcore模块中

```java
    @Override
    public String findLibrary(String name) {
        return pathList.findLibrary(name);
    }
```

查看pathList的定义,实际上调用的是DexPathList的findLibrary接口

```
    /** structured lists of path elements */
    private final DexPathList pathList;

    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);

        this.originalPath = dexPath;
        this.originalLibraryPath = libraryPath;
        
        //在构造函数中进行初始化的
        this.pathList =
            new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
```

### [DexPathList.java]

[DexPathList.java]

```java
    /**
     * Finds the named native code library on any of the library
     * directories pointed at by this instance. This will find the
     * one in the earliest listed directory, ignoring any that are not
     * readable regular files.
     *
     * @return the complete path to the library or {@code null} if no
     * library was found
     */
    public String findLibrary(String libraryName) {
        //fileName会变成 "lib" + libraryName + ".so"
        //这也是为什么我们system.loadLibrary不需要填写lib和so
        String fileName = System.mapLibraryName(libraryName);
        
        //从可能存放so的文件夹列表路径中进行查找
        for (File directory : nativeLibraryDirectories) {
            //判断so文件是否存在该路径下
            File file = new File(directory, fileName);
            if (file.exists() && file.isFile() && file.canRead()) {
                //如果是则返回so文件的绝对路径
                return file.getPath();
            }
        }

        return null;
    }
```

这边可以看出nativeLibraryDirectories其实就是一个File[]数组，存放的是可能存在so文件的文件夹路径，包含应用本身的lib和系统的lib路径

```java
    /** list of native library directory elements */
    private final File[] nativeLibraryDirectories;
    
    /**
     * Constructs an instance.
     *
     * @param definingContext the context in which any as-yet unresolved
     * classes should be defined
     * @param dexPath list of dex/resource path elements, separated by
     * {@code File.pathSeparator}
     * @param libraryPath list of native library directory path elements,
     * separated by {@code File.pathSeparator}
     * @param optimizedDirectory directory where optimized {@code .dex} files
     * should be found and written to, or {@code null} to use the default
     * system directory for same
     */
    public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {
        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }

        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
        }

        if (optimizedDirectory != null) {
            if (!optimizedDirectory.exists())  {
                throw new IllegalArgumentException(
                        "optimizedDirectory doesn't exist: "
                        + optimizedDirectory);
            }

            if (!(optimizedDirectory.canRead()
                            && optimizedDirectory.canWrite())) {
                throw new IllegalArgumentException(
                        "optimizedDirectory not readable/writable: "
                        + optimizedDirectory);
            }
        }

        this.definingContext = definingContext;
        this.dexElements =
            makeDexElements(splitDexPath(dexPath), optimizedDirectory);
        //这边便是nativeLibraryDirectories的赋值操作
        this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
    }
```

splitLibraryPath获取系统java虚拟机默认lib路径列表，并且和传入的path(class loader's library path)合并成一个File[]数组返回。

- 其中class loader's library path一般为/data/app/${package-name}/lib/arm/
- 而System.getProperty("java.library.path", ".")是从系统配置中获取VM的library path，一般可能为/vendor/lib:/system/lib，其实多个目录用:分隔，64位系统可能是/vendor/lib64:/system/lib64 (不同机器有所差别，不一定是固定2个目录);
- splitLibraryPath就是把上述多个可能存在so的目录分隔出来，并存放在同一个数组返回;
- 并且应用的lib目录会放在第一个位置，也就是说so文件会先从应用本身的目录取查找，不存在的话再去查找系统的目录;

```java
/**
     * Splits the given library directory path string into elements
     * using the path separator ({@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android, appending on the elements
     * from the system library path, and pruning out any elements that
     * do not refer to existing and readable directories.
     */
    private static File[] splitLibraryPath(String path) {
        /*
         * Native libraries may exist in both the system and
         * application library paths, and we use this search order:
         *
         *   1. this class loader's library path for application
         *      libraries
         *   2. the VM's library path from the system
         *      property for system libraries
         *
         * This order was reversed prior to Gingerbread; see http://b/2933456.
         */
        ArrayList<File> result = splitPaths(
                path, System.getProperty("java.library.path", "."), true);
        return result.toArray(new File[result.size()]);
    }
复制代码
```

splitPaths作用是将path1和path1通过splitAndAdd分隔成一个个合适的路径，添加在resultList中返回

```java
    /**
     * Splits the given path strings into file elements using the path
     * separator, combining the results and filtering out elements
     * that don't exist, aren't readable, or aren't either a regular
     * file or a directory (as specified). Either string may be empty
     * or {@code null}, in which case it is ignored. If both strings
     * are empty or {@code null}, or all elements get pruned out, then
     * this returns a zero-element list.
     */
    private static ArrayList<File> splitPaths(String path1, String path2,
            boolean wantDirectories) {
        ArrayList<File> result = new ArrayList<File>();

        splitAndAdd(path1, wantDirectories, result);
        splitAndAdd(path2, wantDirectories, result);
        return result;
    }
```

splitAndAdd方法的作用只是将path按":"分隔成一个个的路径，添加在resultList中返回

```java
    /**
     * Helper for {@link #splitPaths}, which does the actual splitting
     * and filtering and adding to a result.
     */
    private static void splitAndAdd(String path, boolean wantDirectories,
            ArrayList<File> resultList) {
        if (path == null) {
            return;
        }
        
        //File.pathSeparator一般为":",这边是将通过:分隔的多个路径分隔出来
        String[] strings = path.split(Pattern.quote(File.pathSeparator));
        
        //判断每个路径下的文件是否符合条件
        for (String s : strings) {
            File file = new File(s);
            //是否存在并且可读
            if (! (file.exists() && file.canRead())) {
                continue;
            }

            /*
             * Note: There are other entities in filesystems than
             * regular files and directories.
             */
             //判断是否要文件夹，还是文件
            if (wantDirectories) {
                if (!file.isDirectory()) {
                    continue;
                }
            } else {
                if (!file.isFile()) {
                    continue;
                }
            }
            //返回符合的文件路径列表
            resultList.add(file);
        }
    }
```

## 查找so文件所在绝对路径(ClassLoader不存在时)

[Runtime.java] java.lang.Runtime

ClassLoader不存在时会尝试从getLibPaths中获取,查看其实现，其实就是返回system指定的vm library path列表，相比ClassLoader存在时，少了个app lib的路径。

```java
    private volatile String[] mLibPaths = null;
    private String[] getLibPaths() {
        if (mLibPaths == null) {
            synchronized(this) {
                if (mLibPaths == null) {
                    mLibPaths = initLibPaths();
                }
            }
        }
        return mLibPaths;
    }
```

initLibPaths的作用就是读取系统配置"java.library.path"指定的vm所需要的library路径列表，同样是通过":"分隔出每一个路径，并返回系统路径列表

```java
    private static String[] initLibPaths() {
        String javaLibraryPath = System.getProperty("java.library.path");
        if (javaLibraryPath == null) {
            return EmptyArray.STRING;
        }
        String[] paths = javaLibraryPath.split(":");
        // Add a '/' to the end of each directory so we don't have to do it every time.
        for (int i = 0; i < paths.length; ++i) {
            if (!paths[i].endsWith("/")) {
                paths[i] += "/";
            }
        }
        return paths;
    }
```

# nativeLoad加载指定路径的so文件

## [Runtime.java] java.lang.Runtime

```java
private static native String nativeLoad(String filename, ClassLoader loader);

```

通过搜索源码会发现调用了OpenjdkJvm.cc下的JVM_NativeLoad方法

## [OpenjdkJvm.cc] 位于platform_art模块

```java
JNIEXPORT jstring JVM_NativeLoad(JNIEnv* env,
                                 jstring javaFilename,
                                 jobject javaLoader,
                                 jclass caller) {
  ScopedUtfChars filename(env, javaFilename);
  if (filename.c_str() == nullptr) {
    return nullptr;
  }

  std::string error_msg;
  {
    art::JavaVMExt* vm = art::Runtime::Current()->GetJavaVM();
    //这边是实际执行so加载的操作调用
    bool success = vm->LoadNativeLibrary(env,
                                         filename.c_str(),
                                         javaLoader,
                                         caller,
                                         &error_msg);
    if (success) {
      return nullptr;
    }
  }

  // Don't let a pending exception from JNI_OnLoad cause a CheckJNI issue with NewStringUTF.
  env->ExceptionClear();
  return env->NewStringUTF(error_msg.c_str());
}
```

实际上调用了Java_vm_ext.cc中的LoadNativeLibrary来实现so加载

## [Java_vm_ext.cc] platform_art/runtime/jni

这部分主要处理如下3件事:

1.判断该so是否已经加载过了，如果符合，不再重复加载； 2.尝试使用OpenNativeLibrary加载该so文件； 3.加载成功后，搜索该so文件里面是否有JNI_OnLoad方法，有的话则主动触发一次调用；

```
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
                                  jobject class_loader,
                                  jclass caller_class,
                                  std::string* error_msg) {
  error_msg->clear();

  //1.这部分判断是否已经加载过该so，并且class loader匹配，是的话返回成功不再处理
  SharedLibrary* library;
  Thread* self = Thread::Current();
  {
    // TODO: move the locking (and more of this logic) into Libraries.
    MutexLock mu(self, *Locks::jni_libraries_lock_);
    //从缓存读取
    library = libraries_->Get(path);
  }
  void* class_loader_allocator = nullptr;
  ....//省略一部分代码
  if (library != nullptr) {
    // Use the allocator pointers for class loader equality to avoid unnecessary weak root decode.
    //判断是否是相同的class loader，不允许so被不同的class loader加载
    if (library->GetClassLoaderAllocator() != class_loader_allocator) {
      ....//省略一部分代码
      return false;
    }

    VLOG(jni) << "[Shared library \"" << path << "\" already loaded in "
              << " ClassLoader " << class_loader << "]";
    if (!library->CheckOnLoadResult()) {
      StringAppendF(error_msg, "JNI_OnLoad failed on a previous attempt "
          "to load \"%s\"", path.c_str());
      return false;
    }
    return true;
  }

  ScopedLocalRef<jstring> library_path(env, GetLibrarySearchPath(env, class_loader));
  

  Locks::mutator_lock_->AssertNotHeld(self);
  const char* path_str = path.empty() ? nullptr : path.c_str();
  bool needs_native_bridge = false;
  char* nativeloader_error_msg = nullptr;
  
  //2. 没加载过该so，需要进行加载
  // 参数patch_str传递的是动态库的全路径，之所以还要传递搜索路径，是因为可能包含它的依赖库
  void* handle = android::OpenNativeLibrary(
      env,
      runtime_->GetTargetSdkVersion(),
      path_str,
      class_loader,
      (caller_location.empty() ? nullptr : caller_location.c_str()),
      library_path.get(),
      &needs_native_bridge,
      &nativeloader_error_msg);
  VLOG(jni) << "[Call to dlopen(\"" << path << "\", RTLD_NOW) returned " << handle << "]";

  if (handle == nullptr) {
    *error_msg = nativeloader_error_msg;
    android::NativeLoaderFreeErrorMessage(nativeloader_error_msg);
    VLOG(jni) << "dlopen(\"" << path << "\", RTLD_NOW) failed: " << *error_msg;
    return false;
  }
  ....//省略一部分代码
  
  //3.加载so成功后，去查看该so是否有"JNI_OnLoad"方法，有的话则进行调用
  void* sym = library->FindSymbol("JNI_OnLoad", nullptr);
  if (sym == nullptr) {
    VLOG(jni) << "[No JNI_OnLoad found in \"" << path << "\"]";
    was_successful = true;
  } else {
    // Call JNI_OnLoad.  We have to override the current class
    // loader, which will always be "null" since the stuff at the
    // top of the stack is around Runtime.loadLibrary().  (See
    // the comments in the JNI FindClass function.)
    ScopedLocalRef<jobject> old_class_loader(env, env->NewLocalRef(self->GetClassLoaderOverride()));
    self->SetClassLoaderOverride(class_loader);

    VLOG(jni) << "[Calling JNI_OnLoad in \"" << path << "\"]";
    using JNI_OnLoadFn = int(*)(JavaVM*, void*);
    JNI_OnLoadFn jni_on_load = reinterpret_cast<JNI_OnLoadFn>(sym);
    int version = (*jni_on_load)(this, nullptr);

    if (IsSdkVersionSetAndAtMost(runtime_->GetTargetSdkVersion(), SdkVersion::kL)) {
      // Make sure that sigchain owns SIGSEGV.
      EnsureFrontOfChain(SIGSEGV);
    }

    self->SetClassLoaderOverride(old_class_loader.get());
    
    //"JNI_OnLoad"方法返回错误
    if (version == JNI_ERR) {
      StringAppendF(error_msg, "JNI_ERR returned from JNI_OnLoad in \"%s\"", path.c_str());
    } 
    //"JNI_OnLoad"方法返回不支持的版本，目前只支持JNI_VERSION_1_2 、 JNI_VERSION_1_4 、 JNI_VERSION_1_6，一般都返回JNI_VERSION_1_6就行
    else if (JavaVMExt::IsBadJniVersion(version)) {
      StringAppendF(error_msg, "Bad JNI version returned from JNI_OnLoad in \"%s\": %d",
                    path.c_str(), version);
    } else {
      was_successful = true;
    }
    VLOG(jni) << "[Returned " << (was_successful ? "successfully" : "failure")
              << " from JNI_OnLoad in \"" << path << "\"]";
  }
  //返回加载结果    
  library->SetResult(was_successful);
  return was_successful;
}
```


