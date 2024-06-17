# 线索
PMS 是什么、启动、安装、解析扫描

![image-20210302220757211](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210302220757.png)

![image-20210628151304180](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708155033.png)





![image-20210708155916472](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708155916.png)


![image-20210708155934107](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708155934.png)


# 概述
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131535911.png)
Android系统启动时，会启动(应用程序管理服务器PKMS)，此服务负责扫描系统中特定的目录，寻 找 里面的APK格式的文件，并对这些文件进行解析，然后得到应用程序相关信息，最后完成应用程序的 安 装

    PKMS在安装应用过程中, 会全面解析应用程序的AndroidManifest.xml文件, 来得到Activity, Service, BroadcastReceiver, ContextProvider 等信息, 在结合PKMS服务就可以在OS中正常的使 用应用程序了

在Android系统中, 系统启动时由SystemServer启动PKMS服务, 启动该服务后会执行应用程序的安装 过程,

接下来后就会重点的介绍 (SystemServer启动PKMS服务的过程, 讲解在Android系统中安装应用程序 的过程)

简单来需知:PKMS 与 AMS 一样，也是Android系统核心服务之一，非常非常的重要，主要完成以下**核心功能**:

1.解析AndroidNanifest.xml清单文件，解析清单文件中的所有节点信息 
2.扫描.apk文件，安装系统应用，安装本地应用等 
3.管理本地应用，主要有， 安装，卸载，应用信息查询 等



研究Android的包管理机制，同样可以从以下几个角度来思考：

1. 被管理的对象是什么？
2. 管理者的职能是什么？
3. 管理机制是如何运转的？

PackageManagerService(简称PKMS)，是Android系统中核心服务之一，**管理着所有跟package相关的工作，常见的比如安装、卸载应用**。 PKMS服务也是通过binder进行通信，IPackageManager.aidl由工具转换后自动生成binder的服务端IPackageManager.Stub和客户端IPackageManager.Stub.Proxy，具体关系如图：



![package_manager_service](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427100730.jpg)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131537758.png)


- Binder服务端：PackageManagerService继承于IPackageManager.Stub；
- Binder客户端：ApplicationPackageManager(简称APM)的成员变量`mPM`继承于IPackageManager.Stub.Proxy; 本身APM是继承于PackageManager对象。

Android系统启动过程中，一路启动到[SystemServer](http://gityuan.com/2016/02/20/android-system-server-2/)后，便可以启动framework的各大服务，本文将介绍PKMS的启动过程

# 启动

`Zygote --> SystemServer --> PackageManagerService(PMS)`

![这里写图片描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427103401.png)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131538217.png)


## PackageManagerService 构造方法
PKMS初始化过程，分为5个阶段：

1. **PMS_START阶段(开始阶段)**：
   - 创建Settings对象；
   - 将6类shareUserId到mSettings；
   - 初始化SystemConfig；
   - 创建名为“PackageManager”的handler线程`mHandlerThread`;
   - 创建UserManagerService多用户管理服务；
   - 通过解析4大目录中的xmL文件构造共享mSharedLibraries；
2. **PMS_SYSTEM_SCAN_START阶段(扫描系统阶段)**：
   - mSharedLibraries共享库中的文件执行dexopt操作；
   - system/framework目录中满足条件的apk或jar文件执行dexopt操作；
   - 扫描系统apk;
3. **PMS_DATA_SCAN_START阶段(扫描Data分区阶段)**：
   - 扫描/data/app目录下的apk;
   - 扫描/data/app-private目录下的apk;
4. **PMS_SCAN_END阶段(扫描结束阶段)**：
   - 将上述信息写回/data/system/packages.xml;
5. **PMS_READY阶段(准备阶段)**：
   - 创建服务PackageInstallerService；


## PMS 启动过程

### 涉及模块

![PMS 涉及到的模块](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427104556.png)



**PMS 启动大致过程如下**

- 1. 和 installed 进行连接，进行安装、卸载操作。
- 1. 创建PackageHandler 线程，处理外部安装、卸载请求。
- 1. 处理系统权限相关的文件`/system/etc/permission/*.xml`。
- 1. 扫描安装jar包 和APK，并获取到安装包的信息


### PMS权限管理(非重点)

![PMS 权限管理](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427104646.png)


### PMS 安装 Jar包 、apk(非重点)

![PMS 安装 Jar包 、apk](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427104723.png)

## APK的扫描
Android 10.0中，PKMS主要扫描以下

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131541471.png)
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131542325.png)
第一步:扫描APK，解析AndroidManifest.xml文件，得到清单文件各个标签内容

第二步:解析清单文件到的信息由 Package 保存。从该类的成员变量可看出，和 Android 四大组件相关 的信息分别由 activites、receivers、providers、services 保存，由于一个 APK 可声明多个组件， 因此activites 和 receivers等均声明为 ArrayList

## APK的安装
安装步骤一: 把Apk的信息通过IO流的形式写入到PackageInstaller.Session中

安装步骤二: 调用PackageInstaller.Session的commit方法, 把Apk的信息交给PKMS处理 安装步骤三: 进行Apk的Copy操作, 进行安装
安装的三步走, 整体描述图:
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131544653.png)
用户点击 xxx.apk 文件进行安装, 从 开始安装 到 完成安装 流程如下:
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131545987.png)

APK的安装, 整体描述图:
摘要：用户点击 xxx.apk 文件进行安装，从 开始安装 到 完成安装 流程如下：
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131546012.png)

### 总结:安装的原理
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131547295.png)

## PMS之权限扫描
此 “PMS之权限扫描” 学习的目标是: PackageManagerService中执行systemReady()后，需求对 /system/etc/permissions中的各种xml进行扫描，进行相应的权限存储，让以后可以使用，这就是本次 “PMS只权限扫描”学习的目的

权限扫描:

PackageManagerService执行systemReady()时，通过SystemConfig的readPermissionsFromXml()来 扫描 读取/system/etc/permissions中的xml文件,包括platform.xml和系统支持的各种硬件模块的

feature主要工作:

整体图:
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303131549485.png)
总结:权限扫描，扫描/system/etc/permissions中的xml，存入相应的结构体中，供之后权限管理 使用


# APK安装流程

## 参考

http://4ch12dy.site/2019/08/01/android-apk-install-process/android-apk-install-process/

https://juejin.cn/post/6844904078015725576#heading-1

https://duanqz.github.io/2017-01-04-Package-Manage-Mechanism#5-apk%E7%9A%84%E5%AE%89%E8%A3%85%E8%BF%87%E7%A8%8B

## PackageInstaller的初始化

### 前言
包管理机制是Android中的重要机制，是应用开发和系统开发需要掌握的知识点之一。
包指的是Apk、jar和so文件等等，它们被加载到Android内存中，由一个包转变成可执行的代码，这就需要一个机制来进行包的加载、解析、管理等操作，这就是包管理机制。包管理机制由许多类一起组成，其中核心为PackageManagerService（PMS）,它负责对包进行管理，如果直接讲PMS会比较难以理解，因此我们需要一个切入点，这个切入点就是常见的APK的安装。
讲到APK的安装之前，先了解下PackageManager、APK文件结构和安装方式。

### 1.PackageManager简介
与ActivityManager和AMS的关系类似，PMS也有一个对应的管理类PackageManager，用于向应用程序进程提供一些功能。PackageManager是一个抽象类，它的具体实现类为ApplicationPackageManager，ApplicationPackageManager中的方法会通过IPackageManager与AMS进行进程间通信，因此PackageManager所提供的功能最终是由PMS来实现的，这么设计的主要用意是为了避免系统服务PMS直接被访问。PackageManager提供了一些功能，主要有以下几点：

1. 获取一个应用程序的所有信息（ApplicationInfo）。
2. 获取四大组件的信息。
3. 查询permission相关信息。
4. 获取包的信息。
5. 安装、卸载APK.

### 2.APK文件结构和安装方式
APK是AndroidPackage的缩写，即Android安装包，它实际上是zip格式的压缩文件，一般情况下，解压后的文件结构如下表所示。

| 目录/文件           | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| assert              | 存放的原生资源文件,通过AssetManager类访问。                  |
| lib                 | 存放库文件。                                                 |
| META-INF            | 保存应用的签名信息，签名信息可以验证APK文件的完整性。        |
| res                 | 存放资源文件。res中除了raw子目录，其他的子目录都参与编译，这些子目录下的资源是通过编译出的R类在代码中访问。 |
| AndroidManifest.xml | 用来声明应用程序的包名称、版本、组件和权限等数据。 apk中的AndroidManifest.xml经过压缩，可以通过AXMLPrinter2工具解开。 |
| classes.dex         | Java源码编译后生成的Java字节码文件。                         |
| resources.arsc      | 编译后的二进制资源文件。                                     |

APK的安装场景主要有以下几种：

- 通过adb命令安装：adb 命令包括adb push/install
- 用户下载的Apk，通过系统安装器packageinstaller安装该Apk。packageinstaller是系统内置的应用程序，用于安装和卸载应用程序。
- 系统开机时安装系统应用。
- 电脑或者手机上的应用商店自动安装。

这4种方式最终都是由PMS来进行处理，在此之前的调用链是不同的，本篇文章会介绍第二种方式，对于用户来说，这是比较常用的安装方式；对于开发者来说，这是调用链比较长的安装方式，能学到的更多。其他的安装场景会在本系列的后续文章进行讲解。

### 3.寻找PackageInstaller入口

在Android7.0之前我们可以通过如下代码安装指定路径中的APK。

```java
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intent.setDataAndType(Uri.parse("file://" + path),"application/vnd.android.package-archive");
context.startActivity(intent);
```

但是Android7.0或更高版本再这么做，就会报FileUriExposedException异常。这是因为StrictMode API 政策禁止应用程序将file:// Uri暴露给另一个应用程序，如果包含file:// Uri的 intent 离开你的应用，就会报FileUriExposedException 异常。为了解决这个问题，谷歌提供了FileProvider，FileProvider继承自ContentProvider ，使用它可以将file://Uri替换为content://Uri，具体怎么使用FileProvider并不是本文的重点，只要知道无论是Android7.0之前还是Android7.0以及更高版本，都会调用如下代码：

```java
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setDataAndType(xxxxx, "application/vnd.android.package-archive");
```

Intent的Action属性为ACTION_VIEW，Type属性指定Intent的数据类型为application/vnd.android.package-archive。
能隐式匹配的Activity为InstallStart，需要注意的是，这里分析的源码基于Android8.0，7.0能隐式匹配的Activity为PackageInstallerActivity。
**packages/apps/PackageInstaller/AndroidManifest.xml**

```xml
<activity android:name=".InstallStart"
               android:exported="true"
               android:excludeFromRecents="true">
           <intent-filter android:priority="1">
               <action android:name="android.intent.action.VIEW" />
               <action android:name="android.intent.action.INSTALL_PACKAGE" />
               <category android:name="android.intent.category.DEFAULT" />
               <data android:scheme="file" />
               <data android:scheme="content" />
               <data android:mimeType="application/vnd.android.package-archive" />
           </intent-filter>
        ...
       </activity>
```

InstallStart是PackageInstaller中的入口Activity，其中PackageInstaller是系统内置的应用程序，用于安装和卸载应用。当我们调用PackageInstaller来安装应用时会跳转到InstallStart，并调用它的onCreate方法：
**packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallStart.java**

```java
@Override
  protected void onCreate(@Nullable Bundle savedInstanceState) {
         if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {//1
          nextActivity.setClass(this, PackageInstallerActivity.class);
      } else {
          Uri packageUri = intent.getData();
          if (packageUri == null) {//2
              Intent result = new Intent();
              result.putExtra(Intent.EXTRA_INSTALL_RESULT,
                      PackageManager.INSTALL_FAILED_INVALID_URI);
              setResult(RESULT_FIRST_USER, result);
              nextActivity = null;
          } else {
              if (packageUri.getScheme().equals(SCHEME_CONTENT)) {//3
                  nextActivity.setClass(this, InstallStaging.class);
              } else {
                  nextActivity.setClass(this, PackageInstallerActivity.class);
              }
          }
      }
      if (nextActivity != null) {
          startActivity(nextActivity);
      }
      finish();
  }
```

注释1处判断Intent的Action是否为CONFIRM_PERMISSIONS，根据本文的应用情景显然不是，接着往下看，注释2处判断packageUri 是否为空也不成立，注释3处，判断Uri的Scheme协议是否是content，如果是就跳转到InstallStaging，如果不是就跳转到PackageInstallerActivity。本文的应用情景中，Android7.0以及更高版本我们会使用FileProvider来处理URI ，FileProvider会隐藏共享文件的真实路径，将路径转换成content://Uri路径，这样就会跳转到InstallStaging。InstallStaging的onResume方法如下所示。

**packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallStaging.java**

```java
@Override
  protected void onResume() {
      super.onResume();
      if (mStagingTask == null) {
          if (mStagedFile == null) {
              try {
                  mStagedFile = TemporaryFileManager.getStagedFile(this);//1
              } catch (IOException e) {
                  showError();
                  return;
              }
          }
          mStagingTask = new StagingAsyncTask();
          mStagingTask.execute(getIntent().getData());//2
      }
  }
```

注释1处如果File类型的mStagedFile 为null，则创建mStagedFile ，mStagedFile用于存储临时数据。 注释2处启动StagingAsyncTask，并传入了content协议的Uri，如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallStaging.java**

```java
 private final class StagingAsyncTask extends AsyncTask<Uri, Void, Boolean> {
        @Override
        protected Boolean doInBackground(Uri... params) {
            if (params == null || params.length <= 0) {
                return false;
            }
            Uri packageUri = params[0];
            try (InputStream in = getContentResolver().openInputStream(packageUri)) {
                if (in == null) {
                    return false;
                }
                try (OutputStream out = new FileOutputStream(mStagedFile)) {
                    byte[] buffer = new byte[4096];
                    int bytesRead;
                    while ((bytesRead = in.read(buffer)) >= 0) {
                        if (isCancelled()) {
                            return false;
                        }
                        out.write(buffer, 0, bytesRead);
                    }
                }
            } catch (IOException | SecurityException e) {
                Log.w(LOG_TAG, "Error staging apk from content URI", e);
                return false;
            }
            return true;
        }
        @Override
        protected void onPostExecute(Boolean success) {
            if (success) {
                Intent installIntent = new Intent(getIntent());
                installIntent.setClass(InstallStaging.this, PackageInstallerActivity.class);
                installIntent.setData(Uri.fromFile(mStagedFile));
                installIntent
                        .setFlags(installIntent.getFlags() & ~Intent.FLAG_ACTIVITY_FORWARD_RESULT);
                installIntent.addFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION);
                startActivityForResult(installIntent, 0);
            } else {
                showError();
            }
        }
    }
}
```

doInBackground方法中将packageUri（content协议的Uri）的内容写入到mStagedFile中，如果写入成功，onPostExecute方法中会跳转到PackageInstallerActivity中，并将mStagedFile传进去。绕了一圈又回到了PackageInstallerActivity，这里可以看出InstallStaging主要起了转换的作用，将content协议的Uri转换为File协议，然后跳转到PackageInstallerActivity，这样就可以像此前版本（Android7.0之前）一样启动安装流程了。

### 4.PackageInstallerActivity解析

从功能上来说，PackageInstallerActivity才是应用安装器PackageInstaller真正的入口Activity，PackageInstallerActivity的onCreate方法如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java**

```java
@Override
protected void onCreate(Bundle icicle) {
    super.onCreate(icicle);
    if (icicle != null) {
        mAllowUnknownSources = icicle.getBoolean(ALLOW_UNKNOWN_SOURCES_KEY);
    }
    mPm = getPackageManager();
    mIpm = AppGlobals.getPackageManager();
    mAppOpsManager = (AppOpsManager) getSystemService(Context.APP_OPS_SERVICE);
    mInstaller = mPm.getPackageInstaller();
    mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);
    ...
    //根据Uri的Scheme进行预处理
    boolean wasSetUp = processPackageUri(packageUri);//1
    if (!wasSetUp) {
        return;
    }
    bindUi(R.layout.install_confirm, false);
    //判断是否是未知来源的应用，如果开启允许安装未知来源选项则直接初始化安装
    checkIfAllowedAndInitiateInstall();//2
}
```

首先初始话安装所需要的各种对象，比如PackageManager、IPackageManager、AppOpsManager和UserManager等等，它们的描述如下表所示。

| 类名             | 描述                                                      |
| ---------------- | --------------------------------------------------------- |
| PackageManager   | 用于向应用程序进程提供一些功能，最终的功能是由PMS来实现的 |
| IPackageManager  | 一个AIDL的接口，用于和PMS进行进程间通信                   |
| AppOpsManager    | 用于权限动态检测,在Android4.3中被引入                     |
| PackageInstaller | 提供安装、升级和删除应用程序功能                          |
| UserManager      | 用于多用户管理                                            |

注释1处的processPackageUri方法如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java**

```java
private boolean processPackageUri(final Uri packageUri) {
     mPackageURI = packageUri;
     final String scheme = packageUri.getScheme();//1
     switch (scheme) {
         case SCHEME_PACKAGE: {
             try {
              ...
         } break;
         case SCHEME_FILE: {
             File sourceFile = new File(packageUri.getPath());//1
             //得到sourceFile的包信息
             PackageParser.Package parsed = PackageUtil.getPackageInfo(this, sourceFile);//2
             if (parsed == null) {
                 Log.w(TAG, "Parse error when parsing manifest. Discontinuing installation");
                 showDialogInner(DLG_PACKAGE_ERROR);
                 setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                 return false;
             }
             //对parsed进行进一步处理得到包信息PackageInfo
             mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                     PackageManager.GET_PERMISSIONS, 0, 0, null,
                     new PackageUserState());//3
             mAppSnippet = PackageUtil.getAppSnippet(this, mPkgInfo.applicationInfo, sourceFile);
         } break;
         default: {
             Log.w(TAG, "Unsupported scheme " + scheme);
             setPmResult(PackageManager.INSTALL_FAILED_INVALID_URI);
             finish();
             return false;
         }
     }
     return true;
 }
```

首先在注释1处得到packageUri的Scheme协议，接着根据这个Scheme协议分别对package协议和file协议进行处理，如果不是这两个协议就会关闭PackageInstallerActivity并return false。我们主要来看file协议的处理，注释1处根据packageUri创建一个新的File。注释2处的内部会用PackageParser的parsePackage方法解析这个File（这个File其实是APK文件），得到APK的包信息Package ，Package包含了该APK的所有信息。注释3处会将Package根据uid、用户状态信息和PackageManager的配置等变量对包信息Package做进一步处理得到PackageInfo。
回到PackageInstallerActivity的onCreate方法的注释2处，checkIfAllowedAndInitiateInstall方法如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java**

```java
private void checkIfAllowedAndInitiateInstall() {
       //判断如果允许安装未知来源或者根据Intent判断得出该APK不是未知来源
       if (mAllowUnknownSources || !isInstallRequestFromUnknownSource(getIntent())) {//1
           //初始化安装
           initiateInstall();//2
           return;
       }
       // 如果管理员限制来自未知源的安装, 就弹出提示Dialog或者跳转到设置界面
       if (isUnknownSourcesDisallowed()) {
           if ((mUserManager.getUserRestrictionSource(UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES,
                   Process.myUserHandle()) & UserManager.RESTRICTION_SOURCE_SYSTEM) != 0) {    
               showDialogInner(DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER);
               return;
           } else {
               startActivity(new Intent(Settings.ACTION_SHOW_ADMIN_SUPPORT_DETAILS));
               finish();
           }
       } else {
           handleUnknownSources();//3
       }
   }
```

注释1处判断允许安装未知来源或者根据Intent判断得出该APK不是未知来源，就调用注释2处的initiateInstall方法来初始化安装。如果管理员限制来自未知源的安装, 就弹出提示Dialog或者跳转到设置界面，否则就调用注释3处的handleUnknownSources方法来处理未知来源的APK。注释2处的initiateInstall方法如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java**

```
JAVA


private void initiateInstall() {
      String pkgName = mPkgInfo.packageName;//1
      String[] oldName = mPm.canonicalToCurrentPackageNames(new String[] { pkgName });
      if (oldName != null && oldName.length > 0 && oldName[0] != null) {
          pkgName = oldName[0];
          mPkgInfo.packageName = pkgName;
          mPkgInfo.applicationInfo.packageName = pkgName;
      }
      try {
          //根据包名获取应用程序信息
          mAppInfo = mPm.getApplicationInfo(pkgName,
                  PackageManager.MATCH_UNINSTALLED_PACKAGES);//2
          if ((mAppInfo.flags&ApplicationInfo.FLAG_INSTALLED) == 0) {
              mAppInfo = null;
          }
      } catch (NameNotFoundException e) {
          mAppInfo = null;
      }
      //初始化安装确认界面
      startInstallConfirm();//3
  }
```

注释1处得到包名，注释2处根据包名获取获取应用程序信息ApplicationInfo。注释3处的startInstallConfirm方法如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java**

```
JAVA


private void startInstallConfirm() {
    //省略初始化界面代码
     ...
     AppSecurityPermissions perms = new AppSecurityPermissions(this, mPkgInfo);//1
     final int N = perms.getPermissionCount(AppSecurityPermissions.WHICH_ALL);
     if (mAppInfo != null) {
         msg = (mAppInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0
                 ? R.string.install_confirm_question_update_system
                 : R.string.install_confirm_question_update;
         mScrollView = new CaffeinatedScrollView(this);
         mScrollView.setFillViewport(true);
         boolean newPermissionsFound = false;
         if (!supportsRuntimePermissions) {
             newPermissionsFound =
                     (perms.getPermissionCount(AppSecurityPermissions.WHICH_NEW) > 0);
             if (newPermissionsFound) {
                 permVisible = true;
                 mScrollView.addView(perms.getPermissionsView(
                         AppSecurityPermissions.WHICH_NEW));//2
             }
         }
     ...
 }
```

startInstallConfirm方法中首先初始化安装确认界面，就是我们平常安装APK时出现的界面，界面上有确认和取消按钮并会列出安装该APK需要访问的系统权限。需要注意的是，不同厂商定制的Android系统会有不同的安装确认界面。
注释1处会创建AppSecurityPermissions，它会提取出APK中权限信息并展示出来，这个负责展示的View是AppSecurityPermissions的内部类PermissionItemView。注释2处调用AppSecurityPermissions的getPermissionsView方法来获取PermissionItemView，并将PermissionItemView添加到CaffeinatedScrollView中，这样安装该APK需要访问的系统权限就可以全部的展示出来了，PackageInstaller的初始化工作就完成了。

### 5.总结

现在来总结下PackageInstaller初始化的过程：

1. 根据Uri的Scheme协议不同，跳转到不同的界面，content协议跳转到InstallStart，其他的跳转到PackageInstallerActivity。本文应用场景中，如果是Android7.0以及更高版本会跳转到InstallStart。
2. InstallStart将content协议的Uri转换为File协议，然后跳转到PackageInstallerActivity。
3. PackageInstallerActivity会分别对package协议和file协议的Uri进行处理，如果是file协议会解析APK文件得到包信息PackageInfo。
4. PackageInstallerActivity中会对未知来源进行处理，如果允许安装未知来源或者根据Intent判断得出该APK不是未知来源，就会初始化安装确认界面，如果管理员限制来自未知源的安装, 就弹出提示Dialog或者跳转到设置界面。



## PackageInstaller安装APK

### 前言

在本系列上一篇文章[Android包管理机制（一）PackageInstaller的初始化](http://liuwangshu.cn/framework/pms/1-packageinstaller-initialize.html)中我们学习了PackageInstaller是如何初始化的，这一篇文章我们接着学习PackageInstaller是如何安装APK的。本系列文章的源码基于Android8.0。

### 1.PackageInstaller中的处理

紧接着上一篇的内容，在PackageInstallerActivity调用startInstallConfirm方法初始化安装确认界面后，这个安装确认界面就会呈现给用户，用户如果想要安装这个应用程序就会点击确定按钮，就会调用PackageInstallerActivity的onClick方法，如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java**

```java
public void onClick(View v) {
        if (v == mOk) {
            if (mOk.isEnabled()) {
                if (mOkCanInstall || mScrollView == null) {
                    if (mSessionId != -1) {
                        mInstaller.setPermissionsResult(mSessionId, true);
                        finish();
                    } else {
                        startInstall();//1
                    }
                } else {
                    mScrollView.pageScroll(View.FOCUS_DOWN);
                }
            }
        } else if (v == mCancel) {
            ...
            finish();
        }
    }
```

onClick方法中分别对确定和取消按钮做处理，主要查看对确定按钮的处理，注释1处调用了startInstall方法：
**packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java**

```java
private void startInstall() {
     Intent newIntent = new Intent();
     newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
             mPkgInfo.applicationInfo);
     newIntent.setData(mPackageURI);//1
     newIntent.setClass(this, InstallInstalling.class);
     String installerPackageName = getIntent().getStringExtra(
             Intent.EXTRA_INSTALLER_PACKAGE_NAME);
     if (mOriginatingURI != null) {
         newIntent.putExtra(Intent.EXTRA_ORIGINATING_URI, mOriginatingURI);
     }
     ...
     if(localLOGV) Log.i(TAG, "downloaded app uri="+mPackageURI);
     startActivity(newIntent);
     finish();
 }
```

startInstall方法用于跳转到InstallInstalling这个Activity，并关闭掉当前的PackageInstallerActivity。InstallInstalling主要用于向包管理器发送包的信息并处理包管理的回调。 InstallInstalling的onCreate方法如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java**

```java
@Override
 protected void onCreate(@Nullable Bundle savedInstanceState) {
     super.onCreate(savedInstanceState);
     setContentView(R.layout.install_installing);
     ApplicationInfo appInfo = getIntent()
             .getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
     mPackageURI = getIntent().getData();
     if ("package".equals(mPackageURI.getScheme())) {
         try {
             getPackageManager().installExistingPackage(appInfo.packageName);
             launchSuccess();
         } catch (PackageManager.NameNotFoundException e) {
             launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
         }
     } else {
     //根据mPackageURI创建一个对应的File 
         final File sourceFile = new File(mPackageURI.getPath());
         PackageUtil.initSnippetForNewApp(this, PackageUtil.getAppSnippet(this, appInfo,
                 sourceFile), R.id.app_snippet);
         //如果savedInstanceState不为null，获取此前保存的mSessionId和mInstallId       
         if (savedInstanceState != null) {//1
             mSessionId = savedInstanceState.getInt(SESSION_ID);
             mInstallId = savedInstanceState.getInt(INSTALL_ID);
           //向InstallEventReceiver注册一个观察者
             try {
                 InstallEventReceiver.addObserver(this, mInstallId,
                         this::launchFinishBasedOnResult);//2
             } catch (EventResultPersister.OutOfIdsException e) {
      
             }
         } else {
             PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                     PackageInstaller.SessionParams.MODE_FULL_INSTALL);//3
             params.referrerUri = getIntent().getParcelableExtra(Intent.EXTRA_REFERRER);
             params.originatingUri = getIntent()
                     .getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
             params.originatingUid = getIntent().getIntExtra(Intent.EXTRA_ORIGINATING_UID,
                     UID_UNKNOWN);
             File file = new File(mPackageURI.getPath());//4
             try {
                 PackageParser.PackageLite pkg = PackageParser.parsePackageLite(file, 0);//5
                 params.setAppPackageName(pkg.packageName);
                 params.setInstallLocation(pkg.installLocation);
                 params.setSize(
                         PackageHelper.calculateInstalledSize(pkg, false, params.abiOverride));
             } catch (PackageParser.PackageParserException e) {
                ...
             }
             try {
                 mInstallId = InstallEventReceiver
                         .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                                 this::launchFinishBasedOnResult);//6
             } catch (EventResultPersister.OutOfIdsException e) {
                 launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
             }
             try {
                 mSessionId = getPackageManager().getPackageInstaller().createSession(params);//7
             } catch (IOException e) {
                 launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
             }
         }
          ...
         mSessionCallback = new InstallSessionCallback();
     }
 }
```

onCreate方法中会分别对package和content协议的Uri进行处理，我们来看content协议的Uri处理部分。注释1处如果savedInstanceState不为null，获取此前保存的mSessionId和mInstallId，其中mSessionId是安装包的会话id，mInstallId是等待的安装事件id。注释2处根据mInstallId向InstallEventReceiver注册一个观察者，launchFinishBasedOnResult会接收到安装事件的回调，无论安装成功或者失败都会关闭当前的Activity(InstallInstalling)。如果savedInstanceState为null，代码的逻辑也是类似的，注释3处创建SessionParams，它用来代表安装会话的参数，注释4、5处根据mPackageUri对包（APK）进行轻量级的解析，并将解析的参数赋值给SessionParams。注释6处和注释2处类似向InstallEventReceiver注册一个观察者返回一个新的mInstallId，其中InstallEventReceiver继承自BroadcastReceiver，用于接收安装事件并回调给EventResultPersister。 注释7处PackageInstaller的createSession方法内部会通过IPackageInstaller与PackageInstallerService进行进程间通信，最终调用的是PackageInstallerService的createSession方法来创建并返回mSessionId。
InstallInstalling的onCreate方法就分析到这，接着查看InstallInstalling的onResume方法：
**packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java**

```java
@Override
 protected void onResume() {
     super.onResume();
     if (mInstallingTask == null) {
         PackageInstaller installer = getPackageManager().getPackageInstaller();
         PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);//1
         if (sessionInfo != null && !sessionInfo.isActive()) {//2
             mInstallingTask = new InstallingAsyncTask();
             mInstallingTask.execute();
         } else {
             mCancelButton.setEnabled(false);
             setFinishOnTouchOutside(false);
         }
     }
 }
```

注释1处根据mSessionId得到SessionInfo，SessionInfo代表安装会话的详细信息。注释2处如果sessionInfo不为Null并且不是活动的，就创建并执行InstallingAsyncTask。InstallingAsyncTask的doInBackground方法中会根据包(APK)的Uri，将APK的信息通过IO流的形式写入到PackageInstaller.Session中。InstallingAsyncTask的onPostExecute方法如下所示。
**packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java**

```java
@Override
 protected void onPostExecute(PackageInstaller.Session session) {
     if (session != null) {
         Intent broadcastIntent = new Intent(BROADCAST_ACTION);
         broadcastIntent.setPackage(
                 getPackageManager().getPermissionControllerPackageName());
         broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);
         PendingIntent pendingIntent = PendingIntent.getBroadcast(
                 InstallInstalling.this,
                 mInstallId,
                 broadcastIntent,
                 PendingIntent.FLAG_UPDATE_CURRENT);
         session.commit(pendingIntent.getIntentSender());//1
         mCancelButton.setEnabled(false);
         setFinishOnTouchOutside(false);
     } else {
         getPackageManager().getPackageInstaller().abandonSession(mSessionId);
         if (!isCancelled()) {
             launchFailure(PackageManager.INSTALL_FAILED_INVALID_APK, null);
         }
     }
 }
```

创建了一个PendingIntent，并将该PendingIntent的IntentSender通过注释1处的PackageInstaller.Session的commit方法发送出去，发送去哪了呢？接着查看PackageInstaller.Session的commit方法。
**frameworks/base/core/java/android/content/pm/PackageInstaller.java**

```java
public void commit(@NonNull IntentSender statusReceiver) {
           try {
               mSession.commit(statusReceiver);
           } catch (RemoteException e) {
               throw e.rethrowFromSystemServer();
           }
       }
```

mSession的类型为IPackageInstallerSession，这说明要通过IPackageInstallerSession来进行进程间的通信，最终会调用PackageInstallerSession的commit方法，这样代码逻辑就到了Java框架层的。

### 2.Java框架层的处理

**frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java**

```java
@Override
   public void commit(IntentSender statusReceiver) {
       Preconditions.checkNotNull(statusReceiver);
       ...
       mActiveCount.incrementAndGet();
       final PackageInstallObserverAdapter adapter = new PackageInstallObserverAdapter(mContext,
               statusReceiver, sessionId, mIsInstallerDeviceOwner, userId);
       mHandler.obtainMessage(MSG_COMMIT, adapter.getBinder()).sendToTarget();//1
   }
```

commit方法中会将包的信息封装为PackageInstallObserverAdapter ，它在PMS中被定义。在注释1处会向Handler发送一个类型为MSG_COMMIT的消息，其中`adapter.getBinder()`会得到IPackageInstallObserver2.Stub类型的观察者，从类型就知道这个观察者是可以跨进程进行回调的。处理该消息的代码如下所示。
**frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java**

```java
private final Handler.Callback mHandlerCallback = new Handler.Callback() {
      @Override
      public boolean handleMessage(Message msg) {
          final PackageInfo pkgInfo = mPm.getPackageInfo(
                  params.appPackageName, PackageManager.GET_SIGNATURES
                          | PackageManager.MATCH_STATIC_SHARED_LIBRARIES /*flags*/, userId);
          final ApplicationInfo appInfo = mPm.getApplicationInfo(
                  params.appPackageName, 0, userId);
          synchronized (mLock) {
              if (msg.obj != null) {
                  mRemoteObserver = (IPackageInstallObserver2) msg.obj;//1
              }
              try {
                  commitLocked(pkgInfo, appInfo);//2
              } catch (PackageManagerException e) {
                  final String completeMsg = ExceptionUtils.getCompleteMessage(e);
                  Slog.e(TAG, "Commit of session " + sessionId + " failed: " + completeMsg);
                  destroyInternal();
                  dispatchSessionFinished(e.error, completeMsg, null);//3
              }
              return true;
          }
      }
  };
```

注释1处获取IPackageInstallObserver2类型的观察者mRemoteObserver，注释2处的commitLocked方法如下所示。
**frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java**

```java
private void commitLocked(PackageInfo pkgInfo, ApplicationInfo appInfo)
          throws PackageManagerException {
     ...
      mPm.installStage(mPackageName, stageDir, stageCid, localObserver, params,
              installerPackageName, installerUid, user, mCertificates);
  }
```

commitLocked方法比较长，这里截取最主要的信息，会调用PMS的installStage方法，这样代码逻辑就进入了PMS中。
回到mHandlerCallback的handleMessage方法，如果commitLocked方法出现PackageManagerException异常，就会调用注释3处的dispatchSessionFinished方法，它的实现如下所示：
**frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java**

```java
private void dispatchSessionFinished(int returnCode, String msg, Bundle extras) {
       mFinalStatus = returnCode;
       mFinalMessage = msg;
       if (mRemoteObserver != null) {
           try {
               mRemoteObserver.onPackageInstalled(mPackageName, returnCode, msg, extras);//1
           } catch (RemoteException ignored) {
           }
       }
       ...
   }
```

注释1处会调用IPackageInstallObserver2的onPackageInstalled方法，具体是实现在PackageInstallObserver类中：
**frameworks/base/core/java/android/app/PackageInstallObserver.java**

```java
public class PackageInstallObserver {
    private final IPackageInstallObserver2.Stub mBinder = new IPackageInstallObserver2.Stub() {
        ...
        @Override
        public void onPackageInstalled(String basePackageName, int returnCode,
                String msg, Bundle extras) {
            PackageInstallObserver.this.onPackageInstalled(basePackageName, returnCode, msg,
                    extras);//1
        }
    };
```

注释1处调用了PackageInstallObserver的onPackageInstalled方法，实现这个方法的类为PackageInstallObserver的子类、前面提到的PackageInstallObserverAdapter。总结一下就是dispatchSessionFinished方法会通过mRemoteObserver的onPackageInstalled方法，将Complete方法出现的PackageManagerException的异常信息回调给PackageInstallObserverAdapter。

### 3.总结

本篇文章讲解了PackageInstaller安装APK的过程，简单来说就两步：

1. 将APK的信息通过IO流的形式写入到PackageInstaller.Session中。
2. 调用PackageInstaller.Session的commit方法，将APK的信息交由PMS处理。




# APK解析流程 [[1-Android插件化开发指南#App的安装流程 3-PMS 启动]]


## PMS处理APK的安装

### 前言

在上一篇文章[Android包管理机制（二）PackageInstaller安装APK](http://liuwangshu.cn/framework/pms/2-packageinstaller-apk.html)中，我们学习了PackageInstaller是如何安装APK的，最后会将APK的信息交由PMS处理。那么PMS是如何处理的呢？这篇文章会给你答案。

### 1.PackageHandler处理安装消息

APK的信息交由PMS后，PMS通过向PackageHandler发送消息来驱动APK的复制和安装工作。
先来查看PackageHandler处理安装消息的调用时序图。

![VeClRJ.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141622.png)

接着上一篇文章的代码逻辑来查看PMS的installStage方法。
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**

```java
void installStage(String packageName, File stagedDir, String stagedCid,
           IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
           String installerPackageName, int installerUid, UserHandle user,
           Certificate[][] certificates) {
       ...
       final Message msg = mHandler.obtainMessage(INIT_COPY);//1
       final int installReason = fixUpInstallReason(installerPackageName, installerUid,
               sessionParams.installReason);
       final InstallParams params = new InstallParams(origin, null, observer,
               sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
               verificationInfo, user, sessionParams.abiOverride,
               sessionParams.grantedRuntimePermissions, certificates, installReason);//2
       params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
       msg.obj = params;
       ...
       mHandler.sendMessage(msg);//3
   }
```

注释2处创建InstallParams，它对应于包的安装数据。注释1处创建了类型为INIT_COPY的消息，在注释3处将InstallParams通过消息发送出去。

#### 1.1 对INIT_COPY的消息的处理

处理INIT_COPY类型的消息的代码如下所示。
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#PackageHandler**

```java
       void doHandleMessage(Message msg) {
           switch (msg.what) {
               case INIT_COPY: {
                   HandlerParams params = (HandlerParams) msg.obj;
                   int idx = mPendingInstalls.size();
                   if (DEBUG_INSTALL) Slog.i(TAG, "init_copy idx=" + idx + ": " + params);
                   //mBound用于标识是否绑定了服务，默认值为false
                   if (!mBound) {//1
                       Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                               System.identityHashCode(mHandler));
                       //如果没有绑定服务，重新绑定，connectToService方法内部如果绑定成功会将mBound置为true
                       if (!connectToService()) {//2
                           Slog.e(TAG, "Failed to bind to media container service");
                           params.serviceError();
                           Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                                   System.identityHashCode(mHandler));
                           if (params.traceMethod != null) {
                               Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, params.traceMethod,
                                       params.traceCookie);
                           }
                           //绑定服务失败则return
                           return;
                       } else {
                           //绑定服务成功，将请求添加到ArrayList类型的mPendingInstalls中，等待处理
                           mPendingInstalls.add(idx, params);
                       }
                   } else {
                   //已经绑定服务
                       mPendingInstalls.add(idx, params);
                       if (idx == 0) {
                           mHandler.sendEmptyMessage(MCS_BOUND);//3
                       }
                   }
                   break;
               }
               ....
               }
   }
}
```

PackageHandler继承自Handler，它被定义在PMS中，doHandleMessage方法用于处理各个类型的消息，来查看对INIT_COPY类型消息的处理。注释1处的mBound用于标识是否绑定了DefaultContainerService，默认值为false。DefaultContainerService是用于检查和复制可移动文件的服务，这是一个比较耗时的操作，因此DefaultContainerService没有和PMS运行在同一进程中，它运行在com.android.defcontainer进程，通过IMediaContainerService和PMS进行IPC通信，如下图所示。
![VeCQG4.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141638.png)

注释2处的connectToService方法用来绑定DefaultContainerService，注释3处发送MCS_BOUND类型的消息，触发处理第一个安装请求。
查看注释2处的connectToService方法：
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#PackageHandler **

```java
private boolean connectToService() {
          if (DEBUG_SD_INSTALL) Log.i(TAG, "Trying to bind to" +
                  " DefaultContainerService");
          Intent service = new Intent().setComponent(DEFAULT_CONTAINER_COMPONENT);
          Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
          if (mContext.bindServiceAsUser(service, mDefContainerConn,
                  Context.BIND_AUTO_CREATE, UserHandle.SYSTEM)) {//1
              Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
              mBound = true;//2
              return true;
          }
          Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
          return false;
      }
```

注释2处如果绑定DefaultContainerService成功，mBound会置为ture 。注释1处的bindServiceAsUser方法会传入mDefContainerConn，bindServiceAsUser方法的处理逻辑和我们调用bindService是类似的，服务建立连接后，会调用onServiceConnected方法：
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java **

```java
class DefaultContainerConnection implements ServiceConnection {
      public void onServiceConnected(ComponentName name, IBinder service) {
          if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceConnected");
          final IMediaContainerService imcs = IMediaContainerService.Stub
                  .asInterface(Binder.allowBlocking(service));
          mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, Object));//1
      }
      public void onServiceDisconnected(ComponentName name) {
          if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceDisconnected");
      }
  }
```

注释1处发送了MCS_BOUND类型的消息，与`PackageHandler.doHandleMessage`方法的注释3处不同的是，这里发送消息带了Object类型的参数，这里会对这两种情况来进行讲解，一种是消息不带Object类型的参数，一种是消息带Object类型的参数。

#### 1.2 对MCS_BOUND类型的消息的处理

**消息不带Object类型的参数**
查看对MCS_BOUND类型消息的处理：

**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**

```java
case MCS_BOUND: {
            if (DEBUG_INSTALL) Slog.i(TAG, "mcs_bound");
            if (msg.obj != null) {//1
                mContainerService = (IMediaContainerService) msg.obj;//2
                Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                        System.identityHashCode(mHandler));
            }
            if (mContainerService == null) {//3
                if (!mBound) {//4
                      Slog.e(TAG, "Cannot bind to media container service");
                      for (HandlerParams params : mPendingInstalls) {
                          params.serviceError();//5
                          Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                        System.identityHashCode(params));
                          if (params.traceMethod != null) {
                          Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER,
                           params.traceMethod, params.traceCookie);
                          }
                          return;
                      }   
                          //绑定失败，清空安装请求队列
                          mPendingInstalls.clear();
                   } else {
                          //继续等待绑定服务
                          Slog.w(TAG, "Waiting to connect to media container service");
                   }
            } else if (mPendingInstalls.size() > 0) {
              ...
              else {
                   Slog.w(TAG, "Empty queue");
                   }
            break;
        }
```

如果消息不带Object类型的参数，就无法满足注释1处的条件，注释2处的IMediaContainerService类型的mContainerService也无法被赋值，这样就满足了注释3处的条件。
如果满足注释4处的条件，说明还没有绑定服务，而此前已经在`PackageHandler.doHandleMessage`方法的注释2处调用绑定服务的方法了，这显然是不正常的，因此在注释5处负责处理服务发生错误的情况。如果不满足注释4处的条件，说明已经绑定服务了，就会打印出系统log，告知用户等待系统绑定服务。

**消息带Object类型的参数**
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**

```java
case MCS_BOUND: {
            if (DEBUG_INSTALL) Slog.i(TAG, "mcs_bound");
            if (msg.obj != null) {
            ...
            }
            if (mContainerService == null) {//1
             ...
            } else if (mPendingInstalls.size() > 0) {//2
                          HandlerParams params = mPendingInstalls.get(0);//3
                        if (params != null) {
                            Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                    System.identityHashCode(params));
                            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "startCopy");
                            if (params.startCopy()) {//4
                                if (DEBUG_SD_INSTALL) Log.i(TAG,
                                        "Checking for more work or unbind...");
                                 //如果APK安装成功，删除本次安装请求
                                if (mPendingInstalls.size() > 0) {
                                    mPendingInstalls.remove(0);
                                }
                                if (mPendingInstalls.size() == 0) {
                                    if (mBound) {
                                    //如果没有安装请求了，发送解绑服务的请求
                                        if (DEBUG_SD_INSTALL) Log.i(TAG,
                                                "Posting delayed MCS_UNBIND");
                                        removeMessages(MCS_UNBIND);
                                        Message ubmsg = obtainMessage(MCS_UNBIND);
                                        sendMessageDelayed(ubmsg, 10000);
                                    }
                                } else {
                                    if (DEBUG_SD_INSTALL) Log.i(TAG,
                                            "Posting MCS_BOUND for next work");
                                   //如果还有其他的安装请求，接着发送MCS_BOUND消息继续处理剩余的安装请求       
                                    mHandler.sendEmptyMessage(MCS_BOUND);//5
                                }
                            }
                            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                        }else {
                        Slog.w(TAG, "Empty queue");//6
                    }
            break;
        }
```

如果MCS_BOUND类型消息带Object类型的参数就不会满足注释1处的条件，就会调用注释2处的判断，如果安装请求数不大于0就会打印出注释6处的log，说明安装请求队列是空的。安装完一个APK后，就会在注释5处发出MSC_BOUND消息，继续处理剩下的安装请求直到安装请求队列为空。
注释3处得到安装请求队列第一个请求HandlerParams ，如果HandlerParams 不为null就会调用注释4处的HandlerParams的startCopy方法，用于开始复制APK的流程。

### 2.复制APK

先来查看复制APK的时序图。

![VeCuIU.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141657.png)

HandlerParams是PMS中的抽象类，它的实现类为PMS的内部类InstallParams。HandlerParams的startCopy方法如下所示。
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#HandlerParams**

```java
final boolean startCopy() {
           boolean res;
           try {
               if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);
               //startCopy方法尝试的次数，超过了4次，就放弃这个安装请求
               if (++mRetries > MAX_RETRIES) {//1
                   Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
                   mHandler.sendEmptyMessage(MCS_GIVE_UP);//2
                   handleServiceError();
                   return false;
               } else {
                   handleStartCopy();//3
                   res = true;
               }
           } catch (RemoteException e) {
               if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
               mHandler.sendEmptyMessage(MCS_RECONNECT);
               res = false;
           }
           handleReturnCode();//4
           return res;
       }
```

注释1处的mRetries用于记录startCopy方法调用的次数，调用startCopy方法时会先自动加1，如果次数大于4次就放弃这个安装请求：在注释2处发送MCS_GIVE_UP类型消息，将第一个安装请求（本次安装请求）从安装请求队列mPendingInstalls中移除掉。注释4处用于处理复制APK后的安装APK逻辑，第3小节中会再次提到它。注释3处调用了抽象方法handleStartCopy，它的实现在InstallParams中，如下所示。
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#InstallParams**

```java
public void handleStartCopy() throws RemoteException {
       ...
       //确定APK的安装位置。onSd：安装到SD卡， onInt：内部存储即Data分区，ephemeral：安装到临时存储（Instant Apps安装）            
       final boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
       final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;
       final boolean ephemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
       PackageInfoLite pkgLite = null;
       if (onInt && onSd) {
         // APK不能同时安装在SD卡和Data分区
           Slog.w(TAG, "Conflicting flags specified for installing on both internal and external");
           ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
         //安装标志冲突，Instant Apps不能安装到SD卡中
       } else if (onSd && ephemeral) {
           Slog.w(TAG,  "Conflicting flags specified for installing ephemeral on external");
           ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
       } else {
            //获取APK的少量的信息
           pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath, installFlags,
                   packageAbiOverride);//1
           if (DEBUG_EPHEMERAL && ephemeral) {
               Slog.v(TAG, "pkgLite for install: " + pkgLite);
           }
       ...
       if (ret == PackageManager.INSTALL_SUCCEEDED) {
            //判断安装的位置
           int loc = pkgLite.recommendedInstallLocation;
           if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
               ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
           } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
               ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
           } 
           ...
           }else{
             loc = installLocationPolicy(pkgLite);//2
             ...
           }
       }
       //根据InstallParams创建InstallArgs对象
       final InstallArgs args = createInstallArgs(this);//3
       mArgs = args;
       if (ret == PackageManager.INSTALL_SUCCEEDED) {
              ...
           if (!origin.existing && requiredUid != -1
                   && isVerificationEnabled(
                         verifierUser.getIdentifier(), installFlags, installerUid)) {
                 ...
           } else{
               ret = args.copyApk(mContainerService, true);//4
           }
       }
       mRet = ret;
   }
```

handleStartCopy方法的代码很多，这里截取关键的部分。
注释1处通过IMediaContainerService跨进程调用DefaultContainerService的getMinimalPackageInfo方法，该方法轻量解析APK并得到APK的少量信息，轻量解析的原因是这里不需要得到APK的全部信息，APK的少量信息会封装到PackageInfoLite中。接着在注释2处确定APK的安装位置。注释3处创建了InstallArgs，InstallArgs 是一个抽象类，定义了APK的安装逻辑，比如复制和重命名APK等，它有3个子类，都被定义在PMS中，如下图所示。

![VeCMiF.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141715.png)

其中FileInstallArgs用于处理安装到非ASEC的存储空间的APK，也就是内部存储空间（Data分区），AsecInstallArgs用于处理安装到ASEC中（mnt/asec）即SD卡中的APK。MoveInstallArgs用于处理已安装APK的移动的逻辑。
对APK进行检查后就会在注释4处调用InstallArgs的copyApk方法进行安装。
不同的InstallArgs子类会有着不同的处理，这里以FileInstallArgs为例。FileInstallArgs的copyApk方法中会直接return FileInstallArgs的doCopyApk方法：
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#FileInstallArgs**

```java
private int doCopyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
        ...
         try {
             final boolean isEphemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
             //创建临时文件存储目录
             final File tempDir =
                     mInstallerService.allocateStageDirLegacy(volumeUuid, isEphemeral);//1
             codeFile = tempDir;
             resourceFile = tempDir;
         } catch (IOException e) {
             Slog.w(TAG, "Failed to create copy file: " + e);
             return PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
         }
         ...
         int ret = PackageManager.INSTALL_SUCCEEDED;
         ret = imcs.copyPackage(origin.file.getAbsolutePath(), target);//2
         ...
         return ret;
     }
```

注释1处用于创建临时存储目录，比如/data/app/vmdl18300388.tmp，其中18300388是安装的sessionId。注释2处通过IMediaContainerService跨进程调用DefaultContainerService的copyPackage方法，这个方法会在DefaultContainerService所在的进程中将APK复制到临时存储目录，比如/data/app/vmdl18300388.tmp/base.apk。目前为止APK的复制工作就完成了，接着就是APK的安装过程了。

### 3.安装APK

照例先来查看安装APK的时序图。

![VeCnaT.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141732.png)

我们回到APK的复制调用链的头部方法：HandlerParams的startCopy方法，在注释4处会调用handleReturnCode方法，它的实现在InstallParams中，如下所示。
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**

```java
 void handleReturnCode() {
    if (mArgs != null) {
        processPendingInstall(mArgs, mRet);
    }
}

    private void processPendingInstall(final InstallArgs args, final int currentStatus) {
        mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);
                PackageInstalledInfo res = new PackageInstalledInfo();
                res.setReturnCode(currentStatus);
                res.uid = -1;
                res.pkg = null;
                res.removedInfo = null;
                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    //安装前处理
                    args.doPreInstall(res.returnCode);//1
                    synchronized (mInstallLock) {
                        installPackageTracedLI(args, res);//2
                    }
                    //安装后收尾
                    args.doPostInstall(res.returnCode, res.uid);//3
                }
              ...
            }
        });
    }
```

handleReturnCode方法中只调用了processPendingInstall方法，注释1处用于检查APK的状态的，在安装前确保安装环境的可靠，如果不可靠会清除复制的APK文件，注释3处用于处理安装后的收尾操作，如果安装不成功，删除掉安装相关的目录与文件。主要来看注释2处的installPackageTracedLI方法，其内部会调用PMS的installPackageLI方法。
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    ...
    PackageParser pp = new PackageParser();
    pp.setSeparateProcesses(mSeparateProcesses);
    pp.setDisplayMetrics(mMetrics);
    pp.setCallback(mPackageParserCallback);
    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
    final PackageParser.Package pkg;
    try {
        //解析APK
        pkg = pp.parsePackage(tmpPackageFile, parseFlags);//1
    } catch (PackageParserException e) {
        res.setError("Failed parse during installPackageLI", e);
        return;
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
    ...
    pp = null;
    String oldCodePath = null;
    boolean systemApp = false;
    synchronized (mPackages) {
        // 检查APK是否存在
        if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
            String oldName = mSettings.getRenamedPackageLPr(pkgName);//获取没被改名前的包名
            if (pkg.mOriginalPackages != null
                    && pkg.mOriginalPackages.contains(oldName)
                    && mPackages.containsKey(oldName)) {
                pkg.setPackageName(oldName);//2
                pkgName = pkg.packageName;
                replace = true;//设置标志位表示是替换安装
                if (DEBUG_INSTALL) Slog.d(TAG, "Replacing existing renamed package: oldName="
                        + oldName + " pkgName=" + pkgName);
            } 
            ...
        }
        PackageSetting ps = mSettings.mPackages.get(pkgName);
        //查看Settings中是否存有要安装的APK的信息，如果有就获取签名信息
        if (ps != null) {//3
            if (DEBUG_INSTALL) Slog.d(TAG, "Existing package: " + ps);
            PackageSetting signatureCheckPs = ps;
            if (pkg.applicationInfo.isStaticSharedLibrary()) {
                SharedLibraryEntry libraryEntry = getLatestSharedLibraVersionLPr(pkg);
                if (libraryEntry != null) {
                    signatureCheckPs = mSettings.getPackageLPr(libraryEntry.apk);
                }
            }
            //检查签名的正确性
            if (shouldCheckUpgradeKeySetLP(signatureCheckPs, scanFlags)) {
                if (!checkUpgradeKeySetLP(signatureCheckPs, pkg)) {
                    res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE, "Package "
                            + pkg.packageName + " upgrade keys do not match the "
                            + "previously installed version");
                    return;
                }
            } 
            ...
        }

        int N = pkg.permissions.size();
        for (int i = N-1; i >= 0; i--) {
           //遍历每个权限，对权限进行处理
            PackageParser.Permission perm = pkg.permissions.get(i);
            BasePermission bp = mSettings.mPermissions.get(perm.info.name);
         
            }
        }
    }
    if (systemApp) {
        if (onExternal) {
            //系统APP不能在SD卡上替换安装
            res.setError(INSTALL_FAILED_INVALID_INSTALL_LOCATION,
                    "Cannot install updates to system apps on sdcard");
            return;
        } else if (instantApp) {
            //系统APP不能被Instant App替换
            res.setError(INSTALL_FAILED_INSTANT_APP_INVALID,
                    "Cannot update a system app with an instant app");
            return;
        }
    }
    ...
    //重命名临时文件
    if (!args.doRename(res.returnCode, pkg, oldCodePath)) {//4
        res.setError(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
        return;
    }

    startIntentFilterVerifications(args.user.getIdentifier(), replace, pkg);

    try (PackageFreezer freezer = freezePackageForInstall(pkgName, installFlags,
            "installPackageLI")) {
       
        if (replace) {//5
         //替换安装   
           ...
            replacePackageLIF(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
                    installerPackageName, res, args.installReason);
        } else {
        //安装新的APK
            installNewPackageLIF(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                    args.user, installerPackageName, volumeUuid, res, args.installReason);
        }
    }

    synchronized (mPackages) {
        final PackageSetting ps = mSettings.mPackages.get(pkgName);
        if (ps != null) {
            //更新应用程序所属的用户
            res.newUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
            ps.setUpdateAvailable(false /*updateAvailable*/);
        }
        ...
    }
}
```

installPackageLI方法的代码有将近500行，这里截取主要的部分，主要做了几件事：

1. 创建PackageParser解析APK。
2. 检查APK是否存在，如果存在就获取此前没被改名前的包名并在注释1处赋值给PackageParser.Package类型的pkg，在注释3处将标志位replace置为true表示是替换安装。
3. 注释3处，如果Settings中保存有要安装的APK的信息，说明此前安装过该APK，则需要校验APK的签名信息，确保安全的进行替换。
4. 在注释4处将临时文件重新命名，比如前面提到的/data/app/vmdl18300388.tmp/base.apk，重命名为/data/app/包名-1/base.apk。这个新命名的包名会带上一个数字后缀1，每次升级一个已有的App，这个数字会不断的累加。
5. 系统APP的更新安装会有两个限制，一个是系统APP不能在SD卡上替换安装，另一个是系统APP不能被Instant App替换。
6. 注释5处根据replace来做区分，如果是替换安装就会调用replacePackageLIF方法，其方法内部还会对系统APP和非系统APP进行区分处理，如果是新安装APK会调用installNewPackageLIF方法。

这里我们以新安装APK为例，会调用PMS的installNewPackageLIF方法。
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**

```java
private void installNewPackageLIF(PackageParser.Package pkg, final int policyFlags,
           int scanFlags, UserHandle user, String installerPackageName, String volumeUuid,
           PackageInstalledInfo res, int installReason) {
       ...
       try {
           //扫描APK
           PackageParser.Package newPackage = scanPackageTracedLI(pkg, policyFlags, scanFlags,
                   System.currentTimeMillis(), user);
           //更新Settings信息
           updateSettingsLI(newPackage, installerPackageName, null, res, user, installReason);
           if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
               //安装成功后，为新安装的应用程序准备数据
               prepareAppDataAfterInstallLIF(newPackage);

           } else {
               //安装失败则删除APK
               deletePackageLIF(pkgName, UserHandle.ALL, false, null,
                       PackageManager.DELETE_KEEP_DATA, res.removedInfo, true, null);
           }
       } catch (PackageManagerException e) {
           res.setError("Package couldn't be installed in " + pkg.codePath, e);
       }
       Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
   }
```

installNewPackageLIF主要做了以下3件事：

1. 扫描APK，将APK的信息存储在PackageParser.Package类型的newPackage中，一个Package的信息包含了1个base APK以及0个或者多个split APK。
2. 更新该APK对应的Settings信息，Settings用于保存所有包的动态设置。
3. 如果安装成功就为新安装的应用程序准备数据，安装失败就删除APK。

安装APK的过程就讲到这里，就不再往下分析下去，有兴趣的同学可以接着深挖。

### 4.总结

本文主要讲解了PMS是如何处理APK安装的，主要有几个步骤：

1. PackageInstaller安装APK时会将APK的信息交由PMS处理，PMS通过向PackageHandler发送消息来驱动APK的复制和安装工作。
2. PMS发送INIT_COPY和MCS_BOUND类型的消息，控制PackageHandler来绑定DefaultContainerService，完成复制APK等工作。
3. 复制APK完成后，会开始进行安装APK的流程，包括安装前的检查、安装APK和安装后的收尾工作。



## APK是如何被解析的
![VeiNCD.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141933.png)

### 前言

在本系列的前面文章中，我介绍了PackageInstaller的初始化和安装APK过程、PMS处理APK的安装和PMS的创建过程，这些文章中经常会涉及到一个类，那就是PackageParser，它用来在APK的安装过程中解析APK，那么APK是如何被解析的呢？这篇文章会给你答案。

### 1.引入PackageParser

Android世界中有很多包，比如应用程序的APK，Android运行环境的JAR包（比如framework.jar）和组成Android系统的各种动态库so等等，由于包的种类和数量繁多，就需要进行包管理，但是包管理需要在内存中进行，而这些包都是以静态文件的形式存在的，就需要一个工具类将这些包转换为内存中的数据结构，这个工具就是包解析器PackageParser。

在[Android包管理机制（三）PMS处理APK的安装](http://liuwangshu.cn/framework/pms/3-pms-install.html)这篇文章中，我们知道安装APK时需要调用PMS的installPackageLI方法：
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    ...
    PackageParser pp = new PackageParser();//1
    pp.setSeparateProcesses(mSeparateProcesses);
    pp.setDisplayMetrics(mMetrics);
    pp.setCallback(mPackageParserCallback);
    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
    final PackageParser.Package pkg;
    try {
        pkg = pp.parsePackage(tmpPackageFile, parseFlags);//2
    }
    ...
 }   
```

可以看到安装APK时，需要先在注释1处创建PackageParser，然后在注释2处调用PackageParser的parsePackage方法来解析APK。

### 2.PackageParser解析APK

Android5.0引入了Split APK机制，这是为了解决65536上限以及APK安装包越来越大等问题。Split APK机制可以将一个APK，拆分成多个独立APK。
在引入了Split APK机制后，APK有两种分类：

- Single APK：安装文件为一个完整的APK，即base APK。Android称其为Monolithic。
- Mutiple APK：安装文件在一个文件目录中，其内部有多个被拆分的APK，这些APK由一个 base APK和一个或多个split APK组成。Android称其为Cluster。

了解了APK，我们接着学习PackageParser解析APK，查看PackageParser的parsePackage方法：
**frameworks/base/core/java/android/content/pm/PackageParser.java**

```java
public Package parsePackage(File packageFile, int flags, boolean useCaches)
           throws PackageParserException {
       Package parsed = useCaches ? getCachedResult(packageFile, flags) : null;
       if (parsed != null) {
           return parsed;
       }
       if (packageFile.isDirectory()) {//1
           parsed = parseClusterPackage(packageFile, flags);
       } else {
           parsed = parseMonolithicPackage(packageFile, flags);
       }
       cacheResult(packageFile, flags, parsed);

       return parsed;
   }
```

注释1处，如果要解析的packageFile是一个目录，说明是Mutiple APK，就需要调用parseClusterPackage方法来解析，如果是Single APK则调用parseMonolithicPackage方法来解析。这里以复杂的parseClusterPackage方法为例，了解了这个方法，parseMonolithicPackage方法自然也看的懂。

![VeiU8e.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141907.png)

**frameworks/base/core/java/android/content/pm/PackageParser.java**

```java

private Package parseClusterPackage(File packageDir, int flags) throws PackageParserException {
        final PackageLite lite = parseClusterPackageLite(packageDir, 0);//1
       if (mOnlyCoreApps && !lite.coreApp) {//2
           throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                   "Not a coreApp: " + packageDir);
       }
       ...
       try {
           final AssetManager assets = assetLoader.getBaseAssetManager();
           final File baseApk = new File(lite.baseCodePath);
           final Package pkg = parseBaseApk(baseApk, assets, flags);//3
           if (pkg == null) {
               throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                       "Failed to parse base APK: " + baseApk);
           }
           if (!ArrayUtils.isEmpty(lite.splitNames)) {
               final int num = lite.splitNames.length;//4
               pkg.splitNames = lite.splitNames;
               pkg.splitCodePaths = lite.splitCodePaths;
               pkg.splitRevisionCodes = lite.splitRevisionCodes;
               pkg.splitFlags = new int[num];
               pkg.splitPrivateFlags = new int[num];
               pkg.applicationInfo.splitNames = pkg.splitNames;
               pkg.applicationInfo.splitDependencies = splitDependencies;
               for (int i = 0; i < num; i++) {
                   final AssetManager splitAssets = assetLoader.getSplitAssetManager(i);
                   parseSplitApk(pkg, i, splitAssets, flags);//5
               }
           }
           pkg.setCodePath(packageDir.getAbsolutePath());
           pkg.setUse32bitAbi(lite.use32bitAbi);
           return pkg;
       } finally {
           IoUtils.closeQuietly(assetLoader);
       }
   }
```

注释1处调用parseClusterPackageLite方法用于轻量级解析目录文件，之所以要轻量级解析是因为解析APK是一个复杂耗时的操作，这里的逻辑并不需要APK所有的信息。parseClusterPackageLite方法内部会通过parseApkLite方法解析每个Mutiple APK，得到每个Mutiple APK对应的ApkLite（轻量级APK信息），然后再将这些ApkLite封装为一个PackageLite（轻量级包信息）并返回。
注释2处，mOnlyCoreApps用来指示PackageParser是否只解析“核心”应用，“核心”应用指的是AndroidManifest中属性coreApp值为true，只解析“核心”应用是为了创建一个极简的启动环境。mOnlyCoreApps在创建PMS时就一路传递过来，如果我们加密了设备，mOnlyCoreApps值就为true，具体的见[Android包管理机制（四）PMS的创建过程](http://liuwangshu.cn/framework/pms/4-pms-start.html)这篇文章的第1小节。另外可以通过PackageParser的setOnlyCoreApps方法来设置mOnlyCoreApps的值。
`lite.coreApp`表示当前包是否包含“核心”应用，如果不满足注释2的条件就会抛出异常。
注释3处的parseBaseApk方法用于解析base APK，注释4处获取split APK的数量，根据这个数量在注释5处遍历调用parseSplitApk来解析每个split APK。这里主要查看parseBaseApk方法，如下所示。
**frameworks/base/core/java/android/content/pm/PackageParser.java**

```java
private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
           throws PackageParserException {
       final String apkPath = apkFile.getAbsolutePath();
       String volumeUuid = null;
       if (apkPath.startsWith(MNT_EXPAND)) {
           final int end = apkPath.indexOf('/', MNT_EXPAND.length());
           volumeUuid = apkPath.substring(MNT_EXPAND.length(), end);//1
       }
       ...
       Resources res = null;
       XmlResourceParser parser = null;
       try {
           res = new Resources(assets, mMetrics, null);
           parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
           final String[] outError = new String[1];
           final Package pkg = parseBaseApk(apkPath, res, parser, flags, outError);//2
           if (pkg == null) {
               throw new PackageParserException(mParseError,
                       apkPath + " (at " + parser.getPositionDescription() + "): " + outError[0]);
           }
           pkg.setVolumeUuid(volumeUuid);//3
           pkg.setApplicationVolumeUuid(volumeUuid);//4
           pkg.setBaseCodePath(apkPath);
           pkg.setSignatures(null);
           return pkg;
       } catch (PackageParserException e) {
           throw e;
       }
       ...
   }
```

注释1处，如果APK的路径以/mnt/expand/开头，就截取该路径获取volumeUuid，注释3处用于以后标识这个解析后的Package，注释4处的用于标识该App所在的存储卷UUID。
注释2处又调用了parseBaseApk的重载方法，可以看出当前的parseBaseApk方法主要是为了获取和设置volumeUuid。parseBaseApk的重载方法如下所示。
**frameworks/base/core/java/android/content/pm/PackageParser.java**

```java

private Package parseBaseApk(String apkPath, Resources res, XmlResourceParser parser, int flags,
           String[] outError) throws XmlPullParserException, IOException {
       ...
       final Package pkg = new Package(pkgName);//1
       //从资源中提取自定义属性集com.android.internal.R.styleable.AndroidManifest得到TypedArray 
       TypedArray sa = res.obtainAttributes(parser,
               com.android.internal.R.styleable.AndroidManifest);//2
       //使用typedarray获取AndroidManifest中的versionCode赋值给Package的对应属性        
       pkg.mVersionCode = pkg.applicationInfo.versionCode = sa.getInteger(
               com.android.internal.R.styleable.AndroidManifest_versionCode, 0);
       pkg.baseRevisionCode = sa.getInteger(
               com.android.internal.R.styleable.AndroidManifest_revisionCode, 0);
       pkg.mVersionName = sa.getNonConfigurationString(
               com.android.internal.R.styleable.AndroidManifest_versionName, 0);
       if (pkg.mVersionName != null) {
           pkg.mVersionName = pkg.mVersionName.intern();
       }
       pkg.coreApp = parser.getAttributeBooleanValue(null, "coreApp", false);//3
       //获取资源后要回收
       sa.recycle();
       return parseBaseApkCommon(pkg, null, res, parser, flags, outError);
   }
```

注释1处创建了Package对象，注释2处从资源中提取自定义属性集 com.android.internal.R.styleable.AndroidManifest得到TypedArray ，这个属性集所在的源码位置为frameworks/base/core/res/res/values/attrs_manifest.xml。接着用TypedArray读取APK的AndroidManifest中的versionCode、revisionCode和versionName的值赋值给Package的对应的属性。
注释3处读取APK的AndroidManifest中的coreApp的值。
最后会调用parseBaseApkCommon方法，这个方法非常长，主要用来解析APK的AndroidManifest中的各个
标签，比如application、permission、uses-sdk、feature-group等等，其中四大组件的标签在application标签下，解析application标签的方法为parseBaseApplication。
**frameworks/base/core/java/android/content/pm/PackageParser.java**

```java
  private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        ...
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
            String tagName = parser.getName();
            if (tagName.equals("activity")) {//1
                Activity a = parseActivity(owner, res, parser, flags, outError, false,
                        owner.baseHardwareAccelerated);//2
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }
                owner.activities.add(a);//3
            } else if (tagName.equals("receiver")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }
                owner.receivers.add(a);
            } else if (tagName.equals("service")) {
                Service s = parseService(owner, res, parser, flags, outError);
                if (s == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }
                owner.services.add(s);
            } else if (tagName.equals("provider")) {
                Provider p = parseProvider(owner, res, parser, flags, outError);
                if (p == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }
                owner.providers.add(p);
             ...
            } 
        }
     ...
}
```

parseBaseApplication方法有近500行代码，这里只截取了解析四大组件相关的代码。注释1处如果标签名为activity，就调用注释2处的parseActivity方法解析activity标签并得到一个Activity对象（PackageParser的静态内部类），这个方法有300多行代码，解析一个activity标签就如此繁琐，activity标签只是Application中众多标签的一个，而Application只是AndroidManifest众多标签的一个，这让我们更加理解了为什么此前解析APK时要使用轻量级解析了。注释3处将解析得到的Activity对象保存在Package的列表activities中。其他的四大组件也是类似的逻辑。
PackageParser解析APK的代码逻辑非常庞大，基本了解本文所讲的就足够了，如果有兴趣可以自行看源码。
parseBaseApk方法主要的解析结构可以理解为以下简图。

![VeiNCD.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141933.png)



### 3.Package的数据结构

包被解析后，最终在内存是Package，Package是PackageParser的内部类，它的部分成员变量如下所示。
**frameworks/base/core/java/android/content/pm/PackageParser.java**

```java
public final static class Package implements Parcelable {
    public String packageName;
    public String manifestPackageName;
    public String[] splitNames;
    public String volumeUuid;
    public String codePath;
    public String baseCodePath;
    ...
    public ApplicationInfo applicationInfo = new ApplicationInfo();
    public final ArrayList<Permission> permissions = new ArrayList<Permission>(0);
    public final ArrayList<PermissionGroup> permissionGroups = new ArrayList<PermissionGroup>(0);
    public final ArrayList<Activity> activities = new ArrayList<Activity>(0);//1
    public final ArrayList<Activity> receivers = new ArrayList<Activity>(0);
    public final ArrayList<Provider> providers = new ArrayList<Provider>(0);
    public final ArrayList<Service> services = new ArrayList<Service>(0);
    public final ArrayList<Instrumentation> instrumentation = new ArrayList<Instrumentation>(0);
...
}
```

注释1处，activities列表中存储了类型为Activity的对象，需要注意的是这个Acticity并不是我们常用的那个Activity，而是PackageParser的静态内部类，Package中的其他列表也都是如此。Package的数据结构简图如下所示。

![Veidvd.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427141951.png)

从这个简图中可以发现Package的数据结构是如何设计的：

- Package中存有许多组件，比如Acticity、Provider、Permission等等，它们都继承基类Component。
- 每个组件都包含一个info数据，比如Activity类中包含了成员变量ActivityInfo，这个ActivityInfo才是真正的Activity数据。
- 四大组件的标签内可能包含`<intent-filter>`来过滤Intent信息，因此需要IntentInfo来保存组件的intent信息，组件基类Component依赖于IntentInfo，IntentInfo有三个子类ActivityIntentInfo、ServiceIntentInfo和ProviderIntentInfo，不同组件依赖的IntentInfo会有所不同，比如Activity继承自`Component<ActivityIntentInfo>` ，Permission继承自`Component<IntentInfo>` 。




# 总结

包管理涉及到的数据结构非常多，在分析源码时，很容易陷入各种数据结构之间的关系，难以自拔，以至于看不到包管理的全貌。作为本文最后的汇总，总结一下各数据结构的职能：

- PackageManagerService # 包管理的核心服务
- com.android.server.pm.Settings # 所有包的管理信息
  - PackageSetting # 每一个包的信息
  - BasePermission # 系统中已有的权限
  - PermissionsState # 授权状态
- PackageParser # 包解析器
  - Package # 解析得到的包信息
  - Component # 组件的基类，其子类对应到<AndroidManifest.xml>中定义的不同组件
    - Activity
    - Provider
    - Service
    - Instrumentation
    - Permission
    - PermissionGroup
  - PackageInfo # 跨进程传递的包数据，包解析时生成
    - PackageItemInfo
      - ApplicationInfo
      - InstrumentationInfo
      - PermissionInfo
      - PermissionGroupInfo
      - ComponentInfo
        - ActivityInfo
        - ServiceInfo
        - ProviderInfo
  - PackageLite # 轻量的包信息
  - ApkLite
- IntentFilter # Intent过滤器
  - IntentInfo # 组件所定义的<intent-filter>信息
    - ActivityIntentInfo
    - ServiceIntentInfo
    - ProviderIntentInfo
- Intent
  - ResolveInfo
  - IntentResolver # Intent解析器，其子类用于不同组件的Intent解析
    - ActivityIntentResolver
    - ServiceIntentResolver
    - ProviderIntentResolver
- PackageHandler # 包管理的消息处理器
  - HandlerParams # 消息的数据载体
    - InstallParams
    - MeasureParams
  - InstallArgs # APK的安装参数
    - FileInstallArgs
    - AsecInstallArgs
    - MoveInfoArgs

如果读者肯花时间同这些繁杂的数据结构周旋，那对于包管理的细节一定可以拿捏的很准确。但猛然一下要理解这么庞大的数据结构设计，实在不是学习包管理机制的上策，毕竟Android也不是一开始就是这么庞大的，譬如包的拆分机制就是较高版本的Android才引入的。随着使用场景的不断丰富，包管理的机制还会更加复杂，建议各位读者还是抓住包管理的几条主线：

- **包扫描的过程**：经过这个过程，Android就能将一个APK文件的静态信息转化为可以管理的数据结构
- **包查询的过程**：Intent的定义和解析是包查询的核心，通过包查询服务可以获取到一个包的信息
- **包安装的过程**：这个过程是包管理者接纳一个新入成员的体现





# 面试题

## apk的安装流程分析

	
## 应用的安装流程	
	


## App解析流程



## Manifest清单文件价值



## apk的打包过程是什么？

aapt 工具打包资源文件，生成 R.java 文件

aidl 工具处理 AIDL 文件，生成对应的 .java 文件

javac 工具编译 Java 文件，生成对应的 .class 文件

把 .class 文件转化成 Davik VM 支持的 .dex 文件

apkbuilder 工具打包生成未签名的 .apk 文件

jarsigner 对未签名 .apk 文件进行签名

zipalign 工具对签名后的 .apk 文件进行对齐处理


## APK  为什么要签名？是否了解过具体的签名机制？



Android 为了确认 apk 开发者身份和防止内容的篡改，设计了一套 apk 签名的方案保证 apk 的安全性，即在打包时由开发者进行 apk 的签名，在安装 apk 时Android 系统会有相应的开发者身份和内容正确性的验证，只有验证通过才可以安装 apk，签名过程和验证的设计就是基于非对称加密的思想。
 Android 在 7.0 以前使用的一套签名方案：在 apk 根目录下的 META-INF/ 文件夹下生成签名文件，然后在安装时在系统的 PackageManagerService 里进行签名文件的验证。
 从 7.0 开始，Android 提供了新的 V2 签名方案：利用 apk(zip) 压缩文件的格式，在几个原始内容区之外增加了一块用于存放签名信息的数据区，然后同样在安装时在系统的 PackageManagerService 里进行 V2 版本的签名验证，V2 方案会更安全、使校验更快安装更快。
 当然 V2 签名方案会向后兼容，如果没有使用 V2 签名就会默认走 V1 签名方案的验证过程。



## 为什么要分 dex ？SDK 21 不分 dex，直接全部加载会不会有什么问题？





## SystemServer处理部分





## 面试题相关

![image-20210413102651345](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210413102651.png)


# 参考

http://gityuan.com/2016/11/06/packagemanager/

https://www.jianshu.com/p/f47e45602ad2

https://duanqz.github.io/2017-01-04-Package-Manage-Mechanism#3-pms%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B

https://github.com/henrymorgen/android-knowledge-system





