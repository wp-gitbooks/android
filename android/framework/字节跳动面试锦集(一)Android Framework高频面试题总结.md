<http://www.blog2019.net/post/170>

1.AMS 、PMS
----------

1.AMS概述

AMS是系统的引导服务，应用进程的启动、切换和调度、四大组件的启动和管理都需要AMS的支持。从这里可以看出AMS的功能会十分的繁多，当然它并不是一个类承担这个重责，它有一些关联类，这在文章后面会讲到。AMS的涉及的知识点非常多，这篇文章主要会讲解AMS的以下几个知识点：

```
AMS的启动流程。
AMS与进程启动。
AMS家族。

```

2.AMS的启动流程
----------

AMS的启动是在SyetemServer进程中启动的，在Android系统启动流程（三）解析SyetemServer进程启动过程这篇文章中提及过，这里从SyetemServer的main方法开始讲起：

```
frameworks/base/services/java/com/android/server/SystemServer.java public static void main(String[] args) {
       new SystemServer().run();
   }
main方法中只调用了SystemServer的run方法，如下所示。
frameworks/base/services/java/com/android/server/SystemServer.java private void run() {
       ...
           System.loadLibrary("android\_servers");//1
       ...
           mSystemServiceManager = new SystemServiceManager(mSystemContext);//2
           LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
       ...    
        try {
           Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartServices");
           startBootstrapServices();//3
           startCoreServices();//4
           startOtherServices();//5
       } catch (Throwable ex) {
           Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*");
           Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\* Failure starting system services", ex);
           throw ex;
       } finally {
           Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
       }
       ...
   }

```

在注释1处加载了动态库libandroid\_servers.so。

接下来在注释2处创建SystemServiceManager，它会对系统的服务进行创建、启动和生命周期管理。在注释3中的startBootstrapServices方法中用SystemServiceManager启动了ActivityManagerService、PowerManagerService、PackageManagerService等服务。

在注释4处的startCoreServices方法中则启动了BatteryService、UsageStatsService和WebViewUpdateService。注释5处的startOtherServices方法中启动了CameraService、AlarmManagerService、VrManagerService等服务。这些服务的父类均为SystemService。

从注释3、4、5的方法可以看出，官方把系统服务分为了三种类型，分别是引导服务、核心服务和其他服务，其中其他服务是一些非紧要和一些不需要立即启动的服务。

系统服务总共大约有80多个，我们主要来查看引导服务AMS是如何启动的，注释3处的startBootstrapServices方法如下所示。

```
frameworks/base/services/java/com/android/server/SystemServer.java private void startBootstrapServices() {
        Installer installer = mSystemServiceManager.startService(Installer.class);
        // Activity manager runs the show.
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();//1
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
      ...
    }

```

在注释1处调用了SystemServiceManager的startService方法，方法的参数是

```
ActivityManagerService.Lifecycle.class：
frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
  @SuppressWarnings("unchecked")
    public <T extends SystemService> T startService(Class\<T\> serviceClass) {
        try {
           ...
            final T service;
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);//1
                service = constructor.newInstance(mContext);//2
            } catch (InstantiationException ex) {
              ...
            }
            // Register it.
            mServices.add(service);//3
            // Start it.
            try {
                service.onStart();//4
            } catch (RuntimeException ex) {
                throw new RuntimeException("Failed to start service " + name
                        + ": onStart threw an exception", ex);
            }
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }

```

startService方法传入的参数是Lifecycle.class，Lifecycle继承自SystemService。

首先，通过反射来创建Lifecycle实例，注释1处得到传进来的Lifecycle的构造器constructor，在注释2处调用constructor的newInstance方法来创建Lifecycle类型的service对象。

接着在注释3处将刚创建的service添加到ArrayList类型的mServices对象中来完成注册。

最后在注释4处调用service的onStart方法来启动service，并返回该service。Lifecycle是AMS的内部类，代码如下所示。

```
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
   public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;
        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);//1
        }
        @Override
        public void onStart() {
            mService.start();//2
        }
        public ActivityManagerService getService() {
            return mService;//3
        }
    }

```

上面的代码结合SystemServiceManager的startService方法来分析，当通过反射来创建Lifecycle实例时，会调用注释1处的方法创建AMS实例，当调用Lifecycle类型的service的onStart方法时，实际上是调用了注释2处AMS的start方法。在SystemServer的startBootstrapServices方法的注释1处，调用了如下代码：

```
 mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();

```

我们知道SystemServiceManager的startService方法最终会返回Lifecycle类型的对象，紧接着又调用了Lifecycle的getService方法，这个方法会返回AMS类型的mService对象，见注释3处，这样AMS实例就会被创建并且返回。

3.AMS与进程启动
----------

在Android系统启动流程（二）解析Zygote进程启动过程这篇文章中，我提到了Zygote的Java框架层中，会创建一个Server端的Socket，这个Socket用来等待AMS来请求Zygote来创建新的应用程序进程。要启动一个应用程序，首先要保证这个应用程序所需要的应用程序进程已经被启动。AMS在启动应用程序时会检查这个应用程序需要的应用程序进程是否存在，不存在就会请求Zygote进程将需要的应用程序进程启动。Service的启动过程中会调用ActiveServices的bringUpServiceLocked方法，如下所示。

```
frameworks/base/services/core/java/com/android/server/am/ActiveServices.java private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg, boolean whileRestarting, boolean permissionsReviewRequired) throws TransactionTooLargeException {
  ...
  final String procName = r.processName;//1
  ProcessRecord app;
  if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);//2
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                        + " app=" + app);
            if (app != null && app.thread != null) {//3
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode,
                    mAm.mProcessStats);
                    realStartServiceLocked(r, app, execInFg);//4
                    return null;
                } catch (TransactionTooLargeException e) {
              ...
            }
        } else {
            app = r.isolatedProc;
        }
 if (app == null && !permissionsReviewRequired) {//5
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {//6
              ...
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }
 ...     }

```

在注释1处得到ServiceRecord的processName的值赋值给procName ，其中ServiceRecord用来描述Service的android:process属性。注释2处将procName和Service的uid传入到AMS的getProcessRecordLocked方法中，来查询是否存在一个与Service对应的ProcessRecord类型的对象app，ProcessRecord主要用来记录运行的应用程序进程的信息。注释5处判断Service对应的app为null则说明用来运行Service的应用程序进程不存在，则调用注释6处的AMS的startProcessLocked方法来创建对应的应用程序进程， 具体的过程请查看Android应用程序进程启动过程（前篇）。

4.AMS家族
-------

ActivityManager是一个和AMS相关联的类，它主要对运行中的Activity进行管理，这些管理工作并不是由ActivityManager来处理的，而是交由AMS来处理，ActivityManager中的方法会通过ActivityManagerNative（以后简称AMN）的getDefault方法来得到ActivityManagerProxy(以后简称AMP)，通过AMP就可以和AMN进行通信，而AMN是一个抽象类，它会将功能交由它的子类AMS来处理，因此，AMP就是AMS的代理类。AMS作为系统核心服务，很多API是不会暴露给ActivityManager的，因此ActivityManager并不算是AMS家族一份子。

为了讲解AMS家族，这里拿Activity的启动过程举例，Activity的启动过程中会调用Instrumentation的execStartActivity方法，如下所示。

```
frameworks/base/core/java/android/app/Instrumentation.java public ActivityResult execStartActivity( Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options) {
      ...
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }

```

execStartActivity方法中会调用AMN的getDefault来获取AMS的代理类AMP。接着调用了AMP的startActivity方法，先来查看AMN的getDefault方法做了什么，如下所示。

```
frameworks/base/core/java/android/app/ActivityManagerNative.java static public IActivityManager getDefault() {
        return gDefault.get();
    }
    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");//1
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);//2
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }+
    };}

```

getDefault方法调用了gDefault的get方法，我们接着往下看，gDefault 是一个Singleton类。注释1处得到名为”activity”的Service引用，也就是IBinder类型的AMS的引用。接着在注释2处将它封装成AMP类型对象，并将它保存到gDefault中，此后调用AMN的getDefault方法就会直接获得AMS的代理对象AMP。

注释2处的asInterface方法如下所示。

```
    frameworks/base/core/java/android/app/ActivityManagerNative.java static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ActivityManagerProxy(obj);}
    asInterface方法的主要作用就是将IBinder类型的AMS引用封装成AMP，AMP的构造方法如下所示。
    frameworks/base/core/java/android/app/ActivityManagerNative.java
    class ActivityManagerProxy implements IActivityManager{
        public ActivityManagerProxy(IBinder remote) {
            mRemote = remote;
        }...
     }

```

AMP的构造方法中将AMS的引用赋值给变量mRemote ，这样在AMP中就可以使用AMS了。 其中IActivityManager是一个接口，AMN和AMP都实现了这个接口，用于实现代理模式和Binder通信。 再回到Instrumentation的execStartActivity方法，来查看AMP的startActivity方法，AMP是AMN的内部类，代码如下所示。

```
frameworks/base/core/java/android/app/ActivityManagerNative.java public int startActivity(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
     ...
       data.writeInt(requestCode);
       data.writeInt(startFlags);
     ...
       mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);//1
       reply.readException();+
       int result = reply.readInt();
       reply.recycle();
       data.recycle();
       return result;
   }

```

首先会将传入的参数写入到Parcel类型的data中。在注释1处，通过IBinder类型对象mRemote（AMS的引用）向服务端的AMS发送一个START\_ACTIVITY\_TRANSACTION类型的进程间通信请求。那么服务端AMS就会从Binder线程池中读取我们客户端发来的数据，最终会调用AMN的onTransact方法，如下所示。

```
frameworks/base/core/java/android/app/ActivityManagerNative.java
   @Override
   public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
       switch (code) {
       case START_ACTIVITY_TRANSACTION:
       {
       ...
           int result = startActivity(app, callingPackage, intent, resolvedType,
                   resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
           reply.writeNoException();
           reply.writeInt(result);
           return true;
       }
   }

```

onTransact中会调用AMS的startActivity方法，如下所示。

```
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
 @Override
 public final int startActivity(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
     return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
             resultWho, requestCode, startFlags, profilerInfo, bOptions,
             UserHandle.getCallingUserId());
 }

```

startActivity方法会最后return startActivityAsUser方法，如下所示。

```
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
 @Override
 public final int startActivityAsUser(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
     enforceNotIsolatedCaller("startActivity");
     userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
             userId, false, ALLOW_FULL_ONLY, "startActivity", null);
     return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
             resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
             profilerInfo, null, null, bOptions, false, userId, null, null);
  }           

```

startActivityAsUser方法最后会return ActivityStarter的startActivityMayWait方法，这一调用过程已经脱离了本节要讲的AMS家族，因此这里不做介绍了，具体的调用过程可以查看Android深入四大组件（一）应用程序启动过程（后篇）这篇文章。

在Activity的启动过程中提到了AMP、AMN和AMS，它们共同组成了AMS家族的主要部分，如下图所示。

AMP是AMN的内部类，它们都实现了IActivityManager接口，这样它们就可以实现代理模式，具体来讲是远程代理：AMP和AMN是运行在两个进程的，AMP是Client端，AMN则是Server端，而Server端中具体的功能都是由AMN的子类AMS来实现的，因此，AMP就是AMS在Client端的代理类。AMN又实现了Binder类，这样AMP可以和AMS就可以通过Binder来进行进程间通信。

ActivityManager通过AMN的getDefault方法得到AMP，通过AMP就可以和AMN进行通信，也就是间接的与AMS进行通信。除了ActivityManager，其他想要与AMS进行通信的类都需要通过AMP，如下图所示。

PMS之前言
------

PMS的创建过程分为两个部分进行讲解，分别是SyetemServer处理部分和PMS构造方法。其中SyetemServer处理部分和AMS和WMS的创建过程是类似的，可以将它们进行对比，这样可以更好的理解和记忆这一知识点。

1PMS之SyetemServer处理部分
------------------------

PMS是在SyetemServer进程中被创建的，SyetemServer进程用来创建系统服务，不了解它的可以查看Android系统启动流程（三）解析SyetemServer进程启动过程这篇文章。 从SyetemServer的入口方法main方法开始讲起，如下所示。

```
frameworks/base/services/java/com/android/server/SystemServer.java public static void main(String[] args) {
        new SystemServer().run();
    }

```

main方法中只调用了SystemServer的run方法，如下所示。

```
frameworks/base/services/java/com/android/server/SystemServer.java private void run() {
    try {
        ...
        //创建消息Looper
         Looper.prepareMainLooper();
        //加载了动态库libandroid\_servers.so
        System.loadLibrary("android\_servers");//1
        performPendingShutdown();
        // 创建系统的Context
        createSystemContext();
        // 创建SystemServiceManager
        mSystemServiceManager = new SystemServiceManager(mSystemContext);//2
        mSystemServiceManager.setRuntimeRestarted(mRuntimeRestart);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        SystemServerInitThreadPool.get();
    } finally {
        traceEnd(); 
    }
    try {
        traceBeginAndSlog("StartServices");
        //启动引导服务
        startBootstrapServices();//3
        //启动核心服务
        startCoreServices();//4
        //启动其他服务
        startOtherServices();//5
        SystemServerInitThreadPool.shutdown();
    } catch (Throwable ex) {
        Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*");
        Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\* Failure starting system services", ex);
        throw ex;
    } finally {
        traceEnd();
    }
    ...}

```

在注释1处加载了动态库libandroid\_servers.so。接下来在注释2处创建SystemServiceManager，它会对系统的服务进行创建、启动和生命周期管理。在注释3中的startBootstrapServices方法中用SystemServiceManager启动了ActivityManagerService、PowerManagerService、PackageManagerService等服务。

在注释4处的startCoreServices方法中则启动了DropBoxManagerService、BatteryService、UsageStatsService和WebViewUpdateService。注释5处的startOtherServices方法中启动了CameraService、AlarmManagerService、VrManagerService等服务。这些服务的父类均为SystemService。

从注释3、4、5的方法可以看出，官方把系统服务分为了三种类型，分别是引导服务、核心服务和其他服务，其中其他服务是一些非紧要和一些不需要立即启动的服务。这些系统服务总共有100多个，我们熟知的AMS属于引导服务，WMS属于其他服务，

本文要讲的PMS属于引导服务，因此这里列出引导服务以及它们的作用，见下表。

```
引导服务	作用
Installer	系统安装apk时的一个服务类，启动完成Installer服务之后才能启动其他的系统服务
ActivityManagerService	负责四大组件的启动、切换、调度。
PowerManagerService	计算系统中和Power相关的计算，然后决策系统应该如何反应
LightsService	管理和显示背光LED
DisplayManagerService	用来管理所有显示设备
UserManagerService	多用户模式管理
SensorService	为系统提供各种感应器服务
PackageManagerService	用来对apk进行安装、解析、删除、卸载等等操作

```

查看启动引导服务的注释3处的startBootstrapServices方法。

```
frameworks/base/services/java/com/android/server/SystemServer.java private void startBootstrapServices() {
    ...
    traceBeginAndSlog("StartPackageManagerService");
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);//1
    mFirstBoot = mPackageManagerService.isFirstBoot();//2
    mPackageManager = mSystemContext.getPackageManager();
    traceEnd();
    ...}

```

注释1处的PMS的main方法主要用来创建PMS，其中最后一个参数mOnlyCore代表是否只扫描系统的目录，它在本篇文章中会出现多次，一般情况下它的值为false。注释2处获取boolean类型的变量mFirstBoot，它用于表示PMS是否首次被启动。mFirstBoot是后续WMS创建时所需要的参数，从这里就可以看出系统服务之间是有依赖关系的，它们的启动顺序不能随意被更改。

2\. PMS构造方法
-----------

PMS的main方法如下所示。

```
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java public static PackageManagerService main(Context context, Installer installer, boolean factoryTest, boolean onlyCore) {
        PackageManagerServiceCompilerMapping.checkProperties();
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        ServiceManager.addService("package", m);
        return m;
    }

```

main方法主要做了两件事，一个是创建PMS对象，另一个是将PMS注册到ServiceManager中。 PMS的构造方法大概有600多行，分为5个阶段，每个阶段会打印出相应的EventLog，EventLog用于打印

Android系统的事件日志。

```
1.BOOT\_PROGRESS\_PMS\_START（开始阶段）

2.BOOT\_PROGRESS\_PMS\_SYSTEM\_SCAN\_START（扫描系统阶段）

3.BOOT\_PROGRESS\_PMS\_DATA\_SCAN\_START（扫描Data分区阶段）

4.BOOT\_PROGRESS\_PMS\_SCAN\_END（扫描结束阶段）

5.BOOT\_PROGRESS\_PMS\_READY（准备阶段）

```

2.1 开始阶段

PMS的构造方法中会获取一些包管理需要属性，如下所示。

```
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java public PackageManagerService(Context context, Installer installer, boolean factoryTest, boolean onlyCore) {
        LockGuard.installLock(mPackages, LockGuard.INDEX_PACKAGES);
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "create package manager");
        //打印开始阶段日志
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis())
        ...
        //用于存储屏幕的相关信息
        mMetrics = new DisplayMetrics();
        //Settings用于保存所有包的动态设置
        mSettings = new Settings(mPackages);
        //在Settings中添加多个默认的sharedUserId
        mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);//1
        mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        ...
        mInstaller = installer;
        //创建Dex优化工具类
        mPackageDexOptimizer = new PackageDexOptimizer(installer, mInstallLock, context,
                "\*dexopt\*");
        mDexManager = new DexManager(this, mPackageDexOptimizer, installer, mInstallLock);
        mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());
        mOnPermissionChangeListeners = new OnPermissionChangeListeners(
                FgThread.get().getLooper());
        getDefaultDisplayMetrics(context, mMetrics);
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "get system config");
        //得到全局系统配置信息。
        SystemConfig systemConfig = SystemConfig.getInstance();
        //获取全局的groupId 
        mGlobalGids = systemConfig.getGlobalGids();
        //获取系统权限
        mSystemPermissions = systemConfig.getSystemPermissions();
        mAvailableFeatures = systemConfig.getAvailableFeatures();
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        mProtectedPackages = new ProtectedPackages(mContext);
        //安装APK时需要的锁，保护所有对installd的访问。
        synchronized (mInstallLock) {//1
        //更新APK时需要的锁，保护内存中已经解析的包信息等内容
        synchronized (mPackages) {//2
            //创建后台线程ServiceThread
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /\*allowIo\*/);
            mHandlerThread.start();
            //创建PackageHandler绑定到ServiceThread的消息队列
            mHandler = new PackageHandler(mHandlerThread.getLooper());//3
            mProcessLoggingHandler = new ProcessLoggingHandler();
            //将PackageHandler添加到Watchdog的检测集中
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);//4

        mDefaultPermissionPolicy = new DefaultPermissionGrantPolicy(this);
        mInstantAppRegistry = new InstantAppRegistry(this);
        //在Data分区创建一些目录
        File dataDir = Environment.getDataDirectory();//5
        mAppInstallDir = new File(dataDir, "app");
        mAppLib32InstallDir = new File(dataDir, "app-lib");
        mAsecInternalPath = new File(dataDir, "app-asec").getPath();
        mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
        //创建多用户管理服务
        sUserManager = new UserManagerService(context, this,
                new UserDataPreparer(mInstaller, mInstallLock, mContext, mOnlyCore), mPackages);
         ...
           mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false))//6
      ...     }

```

在开始阶段中创建了很多PMS中的关键对象并赋值给PMS中的成员变量，下面简单介绍这些成员变量。 mSettings ：用于保存所有包的动态设置。注释1处将系统进程的sharedUserId添加到Settings中，sharedUserId用于进程间共享数据，比如两个App的之间的数据是不共享的，如果它们有了共同的sharedUserId，就可以运行在同一个进程中共享数据。

mInstaller ：Installer继承自SystemService，和PMS、AMS一样是系统的服务（虽然名称不像是服务），PMS很多的操作都是由Installer来完成的，比如APK的安装和卸载。在Installer内部，通过IInstalld和installd进行Binder通信，由位于nativie层的installd来完成具体的操作。

systemConfig：用于得到全局系统配置信息。比如系统的权限就可以通过SystemConfig来获取。 mPackageDexOptimizer ： Dex优化的工具类。

mHandler（PackageHandler类型） ：PackageHandler继承自Handler，在注释3处它绑定了后台线程ServiceThread的消息队列。PMS通过PackageHandler驱动APK的复制和安装工作，具体的请看在Android包管理机制（三）PMS处理APK的安装这篇文章。

PackageHandler处理的消息队列如果过于繁忙，有可能导致系统卡住， 因此在注释4处将它添加到Watchdog的监测集中。

Watchdog主要有两个用途，一个是定时检测系统关键服务（AMS和WMS等）是否可能发生死锁，还有一个是定时检测线程的消息队列是否长时间处于工作状态（可能阻塞等待了很长时间）。如果出现上述问题，Watchdog会将日志保存起来，必要时还会杀掉自己所在的进程，也就是SystemServer进程。

sUserManager（UserManagerService类型） ：多用户管理服务。

除了创建这些关键对象，在开始阶段还有一些关键代码需要去讲解：

注释1处和注释2处加了两个锁，其中mInstallLock是安装APK时需要的锁，保护所有对installd的访问；

mPackages是更新APK时需要的锁，保护内存中已经解析的包信息等内容。

注释5处后的代码创建了一些Data分区中的子目录，比如/data/app。

注释6处会解析packages.xml等文件的信息，保存到Settings的对应字段中。packages.xml中记录系统中所有安装的应用信息，包括基本信息、签名和权限。如果packages.xml有安装的应用信息，那么注释6处Settings的readLPw方法会返回true，mFirstBoot的值为false，说明PMS不是首次被启动。

2.2 扫描系统阶段

```
...public PackageManagerService(Context context, Installer installer, boolean factoryTest, boolean onlyCore) {...
            //打印扫描系统阶段日志
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                    startTime);
            ...
            //在/system中创建framework目录
            File frameworkDir = new File(Environment.getRootDirectory(), "framework");
            ...
            //扫描/vendor/overlay目录下的文件
            scanDirTracedLI(new File(VENDOR_OVERLAY_DIR), mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_TRUSTED_OVERLAY, scanFlags | SCAN_TRUSTED_OVERLAY, 0);
            mParallelPackageParserCallback.findStaticOverlayPackages();
            //扫描/system/framework 目录下的文件
            scanDirTracedLI(frameworkDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED,
                    scanFlags | SCAN_NO_DEX, 0);
            final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
            //扫描 /system/priv-app 目录下的文件
            scanDirTracedLI(privilegedAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);
            final File systemAppDir = new File(Environment.getRootDirectory(), "app");
            //扫描/system/app 目录下的文件
            scanDirTracedLI(systemAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
            File vendorAppDir = new File("/vendor/app");
            try {
                vendorAppDir = vendorAppDir.getCanonicalFile();
            } catch (IOException e) {
                // failed to look up canonical path, continue with original one
            }
            //扫描 /vendor/app 目录下的文件
            scanDirTracedLI(vendorAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

       //扫描/oem/app 目录下的文件
        final File oemAppDir = new File(Environment.getOemDirectory(), "app");
        scanDirTracedLI(oemAppDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

        //这个列表代表有可能有升级包的系统App
        final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<String>();//1
        if (!mOnlyCore) {
            Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
            while (psit.hasNext()) {
                PackageSetting ps = psit.next();                 
                if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                    continue;
                }
                //这里的mPackages的是PMS的成员变量，代表scanDirTracedLI方法扫描上面那些目录得到的 
                final PackageParser.Package scannedPkg = mPackages.get(ps.name);
                if (scannedPkg != null) {           
                    if (mSettings.isDisabledSystemPackageLPr(ps.name)) {//2
                       ...
                        //将这个系统App的PackageSetting从PMS的mPackages中移除
                        removePackageLI(scannedPkg, true);
                        //将升级包的路径添加到mExpectingBetter列表中
                        mExpectingBetter.put(ps.name, ps.codePath);
                    }
                    continue;
                }

                if (!mSettings.isDisabledSystemPackageLPr(ps.name)) {
                   ...   
                } else {
                    final PackageSetting disabledPs = mSettings.getDisabledSystemPkgLPr(ps.name);
                    //这个系统App升级包信息在mDisabledSysPackages中,但是没有发现这个升级包存在
                    if (disabledPs.codePath == null || !disabledPs.codePath.exists()) {//5
                        possiblyDeletedUpdatedSystemApps.add(ps.name);//
                    }
                }
            }
        }
        ...        }

```

/system可以称作为System分区，里面主要存储谷歌和其他厂商提供的Android系统相关文件和框架。

Android系统架构分为应用层、应用框架层、系统运行库层（Native 层）、硬件抽象层（HAL层）和Linux内核层，除了Linux内核层在Boot分区，其他层的代码都在System分区。下面列出 System分区的部分子目录。

上面的代码还涉及到/vendor 目录，它用来存储厂商对Android系统的定制部分。

系统扫描阶段的主要工作有以下3点：

1.创建/system的子目录，比如/system/framework、/system/priv-app和/system/app等等

2.扫描系统文件，比如/vendor/overlay、/system/framework、/system/app等等目录下的文件。

3.对扫描到的系统文件做后续处理。

主要来说第3点，一次OTA升级对于一个系统App会有三种情况：

```
这个系统APP无更新。

这个系统APP有更新。

新的OTA版本中，这个系统APP已经被删除。

```

当系统App升级，PMS会将该系统App的升级包设置数据（PackageSetting）存储到Settings的mDisabledSysPackages列表中（具体见PMS的replaceSystemPackageLIF方法），mDisabledSysPackages的类型为ArrayMap\<String, PackageSetting\>。

mDisabledSysPackages中的信息会被PMS保存到packages.xml中的标签下（具体见Settings的writeDisabledSysPackageLPr方法）。

注释2处说明这个系统App有升级包，那么就将该系统App的PackageSetting从mDisabledSysPackages列表中移除，并将系统App的升级包的路径添加到mExpectingBetter列表中，mExpectingBetter的类型为ArrayMap\<String, File\>等待后续处理。

注释5处如果这个系统App的升级包信息存储在mDisabledSysPackages列表中，但是没有发现这个升级包存在，则将它加入到possiblyDeletedUpdatedSystemApps列表中，意为“系统App的升级包可能被删除”，之所以是“可能”，是因为系统还没有扫描Data分区，只能暂放到possiblyDeletedUpdatedSystemApps列表中，等到扫描完Data分区后再做处理。

2.3 扫描Data分区阶段

```
public PackageManagerService(Context context, Installer installer, boolean factoryTest, boolean onlyCore) {
    ...        
    mSettings.pruneSharedUsersLPw();
    //如果不是只扫描系统的目录，那么就开始扫描Data分区。
    if (!mOnlyCore) {
        //打印扫描Data分区阶段日志
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                SystemClock.uptimeMillis());
        //扫描/data/app目录下的文件 
        scanDirTracedLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
        //扫描/data/app-private目录下的文件 
        scanDirTracedLI(mDrmAppPrivateInstallDir, mDefParseFlags
                | PackageParser.PARSE_FORWARD_LOCK,
                scanFlags | SCAN_REQUIRE_KNOWN, 0);
        //扫描完Data分区后，处理possiblyDeletedUpdatedSystemApps列表
        for (String deletedAppName : possiblyDeletedUpdatedSystemApps) {
            PackageParser.Package deletedPkg = mPackages.get(deletedAppName);
            // 从mSettings.mDisabledSysPackages变量中移除去此应用
            mSettings.removeDisabledSystemPackageLPw(deletedAppName);
            String msg;
          //1：如果这个系统App的包信息不在PMS的变量mPackages中，说明是残留的App信息，后续会删除它的数据。
            if (deletedPkg == null) {
                msg = "Updated system package " + deletedAppName
                        + " no longer exists; it's data will be wiped";
                // Actual deletion of code and data will be handled by later
                // reconciliation step
            } else {
            //2：如果这个系统App在mPackages中，说明是存在于Data分区，不属于系统App，那么移除其系统权限。
                msg = "Updated system app + " + deletedAppName
                        + " no longer present; removing system privileges for "
                        + deletedAppName;
                deletedPkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_SYSTEM;
                PackageSetting deletedPs = mSettings.mPackages.get(deletedAppName);
                deletedPs.pkgFlags &= ~ApplicationInfo.FLAG_SYSTEM;
            }
            logCriticalInfo(Log.WARN, msg);
        }
         //遍历mExpectingBetter列表
        for (int i = 0; i < mExpectingBetter.size(); i++) {
            final String packageName = mExpectingBetter.keyAt(i);
            if (!mPackages.containsKey(packageName)) {
                //得到系统App的升级包路径
                final File scanFile = mExpectingBetter.valueAt(i);
                logCriticalInfo(Log.WARN, "Expected better " + packageName
                        + " but never showed up; reverting to system");
                int reparseFlags = mDefParseFlags;
                //3：根据系统App所在的目录设置扫描的解析参数
                if (FileUtils.contains(privilegedAppDir, scanFile)) {
                    reparseFlags = PackageParser.PARSE_IS_SYSTEM
                            | PackageParser.PARSE_IS_SYSTEM_DIR
                            | PackageParser.PARSE_IS_PRIVILEGED;
                } 
                ...
                //将packageName对应的包设置数据（PackageSetting）添加到mSettings的mPackages中
                mSettings.enableSystemPackageLPw(packageName);//4
                try {
                    //扫描系统App的升级包
                    scanPackageTracedLI(scanFile, reparseFlags, scanFlags, 0, null);//5
                } catch (PackageManagerException e) {
                    Slog.e(TAG, "Failed to parse original system package: "
                            + e.getMessage());
                }
            }
        }
    }
   //清除mExpectingBetter列表
    mExpectingBetter.clear();...}

```

/data可以称为Data分区，它用来存储所有用户的个人数据和配置文件。下面列出Data分区部分子目录： 扫描Data分区阶段主要做了以下几件事：

```
1.扫描/data/app和/data/app-private目录下的文件。

2.遍历possiblyDeletedUpdatedSystemApps列表，注释1处如果这个系统App的包信息不在PMS的变量mPackages中，说明是残留的App信息，后续会删除它的数据。注释2处如果这个系统App的包信息在mPackages中，说明是存在于Data分区，不属于系统App，那么移除其系统权限。

3.遍历mExpectingBetter列表，注释3处根据系统App所在的目录设置扫描的解析参数，注释4处的方法内部会将packageName对应的包设置数据（PackageSetting）添加到mSettings的mPackages中。注释5处扫描系统App的升级包，最后清除mExpectingBetter列表。

```

2.4 扫描结束阶段

```
  //打印扫描结束阶段日志
  EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());
            Slog.i(TAG, "Time to scan packages: "
                    + ((SystemClock.uptimeMillis()-startTime)/1000f)
                    + " seconds");
            int updateFlags = UPDATE_PERMISSIONS_ALL;
            // 如果当前平台SDK版本和上次启动时的SDK版本不同，重新更新APK的授权
            if (ver.sdkVersion != mSdkVersion) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
            updatePermissionsLPw(null, null, StorageManager.UUID_PRIVATE_INTERNAL, updateFlags);
            ver.sdkVersion = mSdkVersion;
           //如果是第一次启动或者是Android M升级后的第一次启动，需要初始化所有用户定义的默认首选App
            if (!onlyCore && (mPromoteSystemApps || mFirstBoot)) {
                for (UserInfo user : sUserManager.getUsers(true)) {
                    mSettings.applyDefaultPreferredAppsLPw(this, user.id);
                    applyFactoryDefaultBrowserLPw(user.id);
                    primeDomainVerificationsLPw(user.id);
                }
            }
           ...
            //OTA后的第一次启动，会清除代码缓存目录。
            if (mIsUpgrade && !onlyCore) {
                Slog.i(TAG, "Build fingerprint changed; clearing code caches");
                for (int i = 0; i < mSettings.mPackages.size(); i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if (Objects.equals(StorageManager.UUID_PRIVATE_INTERNAL, ps.volumeUuid)) {
                        clearAppDataLIF(ps.pkg, UserHandle.USER_ALL,
                                StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE
                                        | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);
                    }
                }
                ver.fingerprint = Build.FINGERPRINT;
            }
            ...
           // 把Settings的内容保存到packages.xml中
            mSettings.writeLPr();
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

```

扫描结束结束阶段主要做了以下几件事：

```
1.如果当前平台SDK版本和上次启动时的SDK版本不同，重新更新APK的授权。

2.如果是第一次启动或者是Android M升级后的第一次启动，需要初始化所有用户定义的默认首选App。

3.OTA升级后的第一次启动，会清除代码缓存目录。

4.把Settings的内容保存到packages.xml中，这样此后PMS再次创建时会读到此前保存的Settings的内容。

```

2.5 准备阶段

```
 EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                SystemClock.uptimeMillis());
    ... 
    mInstallerService = new PackageInstallerService(context, this);//1
    ...
    Runtime.getRuntime().gc();//2
    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "loadFallbacks");
    FallbackCategoryProvider.loadFallbacks();
    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    mInstaller.setWarnIfHeld(mPackages);
    LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());//3
    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);}

```

注释1处创建PackageInstallerService，PackageInstallerService是用于管理安装会话的服务，它会为每次安装过程分配一个SessionId，在Android包管理机制（二）PackageInstaller安装APK这篇文章中提到过PackageInstallerService。

注释2处进行一次垃圾收集。注释3处将PackageManagerInternalImpl（PackageManager的本地服务）添加到LocalServices中，LocalServices用于存储运行在当前的进程中的本地服务。

2.Activity 启动流程，App 启动流程
------------------------

Activity的启动模式

```
1.standard:默认标准模式，每启动一个都会创建一个实例，
2.singleTop：栈顶复用，如果在栈顶就调用onNewIntent复用，从onResume()开始
3.singleTask：栈内复用，本栈内只要用该类型Activity就会将其顶部的activity出栈
4.singleInstance：单例模式，除了3中特性，系统会单独给该Activity创建一个栈，

```

1.什么是Zygote进程
-------------

1.1 简单介绍

Zygote进程是所有的android进程的父进程，包括SystemServer和各种应用进程都是通过Zygote进程fork出来的。Zygote（孵化）进程相当于是android系统的根进程，后面所有的进程都是通过这个进程fork出来的

虽然Zygote进程相当于Android系统的根进程，但是事实上它也是由Linux系统的init进程启动的。

1.2 各个进程的先后顺序

init进程 --\> Zygote进程 --\> SystemServer进程 --\>各种应用进程

1.3 进程作用说明

init进程：linux的根进程，android系统是基于linux系统的，因此可以算作是整个android操作系统的第一个进程；

Zygote进程：android系统的根进程，主要作用：可以作用Zygote进程fork出SystemServer进程和各种应用进程；

SystemService进程：主要是在这个进程中启动系统的各项服务，比如ActivityManagerService，PackageManagerService，WindowManagerService服务等等；

各种应用进程：启动自己编写的客户端应用时，一般都是重新启动一个应用进程，有自己的虚拟机与运行环境；

2.Zygote进程的启动流程
---------------

2.1 源码位置

位置：frameworks/base/core/java/com/android/internal/os/ZygoteInit.java

Zygote进程mian方法主要执行逻辑：

初始化DDMS；

注册Zygote进程的socket通讯；

初始化Zygote中的各种类，资源文件，OpenGL，类库，Text资源等等；

初始化完成之后fork出SystemServer进程；

fork出SystemServer进程之后，关闭socket连接；

2.2 ZygoteInit类的main方法

init进程在启动Zygote进程时一般都会调用ZygoteInit类的main方法，因此这里看一下该方法的具体实现(基于android23源码)；

调用enableDdms()，设置DDMS可用，可以发现DDMS启动的时机还是比较早的，在整个Zygote进程刚刚开始要启动额时候就设置可用。

之后初始化各种参数

通过调用registerZygoteSocket方法，注册为Zygote进程注册Socket

然后调用preload方法实现预加载各种资源

然后通过调用startSystemServer开启SystemServer服务，这个是重点

```
public static void main(String argv[]) {
    try {
        //设置ddms可以用
        RuntimeInit.enableDdms();
        SamplingProfilerIntegration.start();
        boolean startSystemServer = false;
        String socketName = "zygote";
        String abiList = null;
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                socketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }

    if (abiList == null) {
        throw new RuntimeException("No ABI list supplied.");
    }

    registerZygoteSocket(socketName);
    EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
        SystemClock.uptimeMillis());
    preload();
    EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
        SystemClock.uptimeMillis());
    SamplingProfilerIntegration.writeZygoteSnapshot();

    gcAndFinalize();
    Trace.setTracingEnabled(false);

    if (startSystemServer) {
        startSystemServer(abiList, socketName);
    }

    Log.i(TAG, "Accepting command socket connections");
    runSelectLoop(abiList);

    closeServerSocket();
} catch (MethodAndArgsCaller caller) {
    caller.run();
} catch (RuntimeException ex) {
    Log.e(TAG, "Zygote died with exception", ex);
    closeServerSocket();
    throw ex;
}

```

} 2.3 registerZygoteSocket(socketName)分析

调用registerZygoteSocket（String socketName）为Zygote进程注册socket

```
private static void registerZygoteSocket(String socketName) {
    if (sServerSocket == null) {
        int fileDesc;
        final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
        try {
            String env = System.getenv(fullSocketName);
            fileDesc = Integer.parseInt(env);
        } catch (RuntimeException ex) {
            throw new RuntimeException(fullSocketName + " unset or invalid", ex);
        }

        try {
            FileDescriptor fd = new FileDescriptor();
            fd.setInt$(fileDesc);
            sServerSocket = new LocalServerSocket(fd);
        } catch (IOException ex) {
            throw new RuntimeException(
                    "Error binding to local socket '" + fileDesc + "'", ex);
        }
    }
}

```

2.4 preLoad()方法分析

源码如下所示

```
static void preload() {
    Log.d(TAG, "begin preload");
    preloadClasses();
    preloadResources();
    preloadOpenGL();
    preloadSharedLibraries();
    preloadTextResources();
    // Ask the WebViewFactory to do any initialization that must run in the zygote process,
    // for memory sharing purposes.
    WebViewFactory.prepareWebViewInZygote();
    Log.d(TAG, "end preload");
}

```

大概操作是这样的：

```
preloadClasses()用于初始化Zygote中需要的class类； preloadResources()用于初始化系统资源； preloadOpenGL()用于初始化OpenGL； preloadSharedLibraries()用于初始化系统libraries； preloadTextResources()用于初始化文字资源； prepareWebViewInZygote()用于初始化webview;

```

2.5 startSystemServer()启动进程

这段逻辑的执行逻辑就是通过Zygote fork出SystemServer进程

```
private static boolean startSystemServer(String abiList, String socketName) throws MethodAndArgsCaller, RuntimeException {
    long capabilities = posixCapabilitiesAsBits(
        OsConstants.CAP_BLOCK_SUSPEND,
        OsConstants.CAP_KILL,
        OsConstants.CAP_NET_ADMIN,
        OsConstants.CAP_NET_BIND_SERVICE,
        OsConstants.CAP_NET_BROADCAST,
        OsConstants.CAP_NET_RAW,
        OsConstants.CAP_SYS_MODULE,
        OsConstants.CAP_SYS_NICE,
        OsConstants.CAP_SYS_RESOURCE,
        OsConstants.CAP_SYS_TIME,
        OsConstants.CAP_SYS_TTY_CONFIG
    );
    /\* Hardcoded command line to start the system server \*/
    String args[] = {
        "--setuid=1000",
        "--setgid=1000",
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
        "--capabilities=" + capabilities + "," + capabilities,
        "--nice-name=system\_server",
        "--runtime-args",
        "com.android.server.SystemServer",
    };
    ZygoteConnection.Arguments parsedArgs = null;

    int pid;

    try {
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        /\* Request to fork the system server process \*/
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

    /\* For child process \*/
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        handleSystemServerProcess(parsedArgs);
    }

    return true;
}

```

3.SystemServer进程启动流程
--------------------

3.1 SystemServer进程简介

SystemServer进程主要的作用是在这个进程中启动各种系统服务，比如ActivityManagerService， PackageManagerService，WindowManagerService服务，以及各种系统性的服务其实都是在SystemServer进程中启动的，而当我们的应用需要使用各种系统服务的时候其实也是通过与SystemServer进程通讯获取各种服务对象的句柄的。

3.2 SystemServer的main方法

如下所示，比较简单，只是new出一个SystemServer对象并执行其run方法，查看SystemServer类的定义我们知道其实final类型的，所以我们一般不能重写或者继承。

3.3 查看run方法

代码如下所示

首先判断系统当前时间，若当前时间小于1970年1月1日，则一些初始化操作可能会处所，所以当系统的当前时间小于1970年1月1日的时候，设置系统当前时间为该时间点。

然后是设置系统的语言环境等

接着设置虚拟机运行内存，加载运行库，设置SystemServer的异步消息

```
private void run() {
    if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
        Slog.w(TAG, "System clock is before 1970; setting to 1970.");
        SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
    }

    if (!SystemProperties.get("persist.sys.language").isEmpty()) {
        final String languageTag = Locale.getDefault().toLanguageTag();

        SystemProperties.set("persist.sys.locale", languageTag);
        SystemProperties.set("persist.sys.language", "");
        SystemProperties.set("persist.sys.country", "");
        SystemProperties.set("persist.sys.localevar", "");
    }

    Slog.i(TAG, "Entered the Android system server!");
    EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, SystemClock.uptimeMillis());

    SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

    if (SamplingProfilerIntegration.isEnabled()) {
        SamplingProfilerIntegration.start();
        mProfilerSnapshotTimer = new Timer();
        mProfilerSnapshotTimer.schedule(new TimerTask() {
            @Override public void run() {
                SamplingProfilerIntegration.writeSnapshot("system\_server", null);
            }
        }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
    }

    // Mmmmmm... more memory!
    VMRuntime.getRuntime().clearGrowthLimit();

    // The system server has to run all of the time, so it needs to be
    // as efficient as possible with its memory usage.
    VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

    // Some devices rely on runtime fingerprint generation, so make sure
    // we've defined it before booting further.
    Build.ensureFingerprintProperty();

    // Within the system server, it is an error to access Environment paths without
    // explicitly specifying a user.
    Environment.setUserRequired(true);

    // Ensure binder calls into the system always run at foreground priority.
    BinderInternal.disableBackgroundScheduling(true);

    // Prepare the main looper thread (this thread).
    android.os.Process.setThreadPriority(
            android.os.Process.THREAD_PRIORITY_FOREGROUND);
    android.os.Process.setCanSelfBackground(false);
    Looper.prepareMainLooper();

    // Initialize native services.
    System.loadLibrary("android\_servers");

    // Check whether we failed to shut down last time we tried.
    // This call may not return.
    performPendingShutdown();

    // Initialize the system context.
    createSystemContext();

    // Create the system service manager.
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

    // Start services.
    try {
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
    } catch (Throwable ex) {
        Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*");
        Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\* Failure starting system services", ex);
        throw ex;
    }

    // For debug builds, log event loop stalls to dropbox for analysis.
    if (StrictMode.conditionallyEnableDebugLogging()) {
        Slog.i(TAG, "Enabled StrictMode for system server main thread.");
    }

    // Loop forever.
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}

```

然后下面的代码是：

```
// Initialize the system context.
createSystemContext();
// Create the system service manager.
mSystemServiceManager = new SystemServiceManager(mSystemContext);
LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
// Start services.try {
    startBootstrapServices();
    startCoreServices();
    startOtherServices();
} catch (Throwable ex) {
    Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*");
    Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\* Failure starting system services", ex);
    throw ex;
}

```

3.4 run方法中createSystemContext()解析

调用createSystemContext()方法：

可以看到在SystemServer进程中也存在着Context对象，并且是通过ActivityThread.systemMain方法创建context的，这一部分的逻辑以后会通过介绍Activity的启动流程来介绍，这里就不在扩展，只知道在SystemServer进程中也需要创建Context对象。

```
private void createSystemContext() {
    ActivityThread activityThread = ActivityThread.systemMain();
    mSystemContext = activityThread.getSystemContext();
    mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
}

```

3.5 mSystemServiceManager的创建

看run方法中，通过SystemServiceManager的构造方法创建了一个新的SystemServiceManager对象，我们知道SystemServer进程主要是用来构建系统各种service服务的，而SystemServiceManager就是这些服务的管理对象。

然后调用：

将SystemServiceManager对象保存SystemServer进程中的一个数据结构中。

LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

最后开始执行：

```
// Start services.try {
    startBootstrapServices();
    startCoreServices();
    startOtherServices();
} catch (Throwable ex) {
    Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*");
    Slog.e("System", "\*\*\*\*\*\*\*\*\*\*\*\* Failure starting system services", ex);
    throw ex;
}

```

里面主要涉及了是三个方法：

```
startBootstrapServices() 主要用于启动系统Boot级服务

startCoreServices() 主要用于启动系统核心的服务

startOtherServices() 主要用于启动一些非紧要或者是非需要及时启动的服务

```

.启动服务

4.1 启动哪些服务

在开始执行启动服务之前总是会先尝试通过socket方式连接Zygote进程，在成功连接之后才会开始启动其他服务。

4.2 启动服务流程源码分析

首先看一下startBootstrapServices方法：

private void startBootstrapServices() { Installer installer = mSystemServiceManager.startService(Installer.class);

```
mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();
mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
mActivityManagerService.setInstaller(installer);

mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

mActivityManagerService.initPowerManagement();

// Manages LEDs and display backlight so we need it to bring up the display.
mSystemServiceManager.startService(LightsService.class);

// Display manager is needed to provide display metrics before package manager
// starts up.
mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

// We need the default display before we can initialize the package manager.
mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

// Only run "core" apps if we're encrypting the device.
String cryptState = SystemProperties.get("vold.decrypt");
if (ENCRYPTING_STATE.equals(cryptState)) {
    Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
    mOnlyCore = true;
} else if (ENCRYPTED_STATE.equals(cryptState)) {
    Slog.w(TAG, "Device encrypted - only parsing core apps");
    mOnlyCore = true;
}

// Start the package manager.
Slog.i(TAG, "Package Manager");
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
        mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
mFirstBoot = mPackageManagerService.isFirstBoot();
mPackageManager = mSystemContext.getPackageManager();

Slog.i(TAG, "User Service");
ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

// Initialize attribute cache used to cache resources from packages.
AttributeCache.init(mSystemContext);

// Set up the Application instance for the system process and get started.
mActivityManagerService.setSystemProcess();

// The sensor service needs access to package manager service, app ops
// service, and permissions service, therefore we start it after them.
startSensorService();

```

}

先执行：

Installer installer = mSystemServiceManager.startService(Installer.class);

mSystemServiceManager是系统服务管理对象，在main方法中已经创建完成，这里我们看一下其

startService方法的具体实现：

可以看到通过反射器构造方法创建出服务类，然后添加到SystemServiceManager的服务列表数据中，最后调用了service.onStart()方法，因为传递的是Installer.class

```
public <T extends SystemService> T startService(Class\<T\> serviceClass) {
    final String name = serviceClass.getName();
    Slog.i(TAG, "Starting " + name);

    // Create the service.
    if (!SystemService.class.isAssignableFrom(serviceClass)) {
        throw new RuntimeException("Failed to create " + name
                + ": service must extend " + SystemService.class.getName());
    }
    final T service;
    try {
        Constructor<T> constructor = serviceClass.getConstructor(Context.class);
        service = constructor.newInstance(mContext);
    } catch (InstantiationException ex) {
        throw new RuntimeException("Failed to create service " + name
                + ": service could not be instantiated", ex);
    } catch (IllegalAccessException ex) {
        throw new RuntimeException("Failed to create service " + name
                + ": service must have a public constructor with a Context argument", ex);
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException("Failed to create service " + name
                + ": service must have a public constructor with a Context argument", ex);
    } catch (InvocationTargetException ex) {
        throw new RuntimeException("Failed to create service " + name
                + ": service constructor threw an exception", ex);
    }

    // Register it.
    mServices.add(service);

    // Start it.
    try {
        service.onStart();
    } catch (RuntimeException ex) {
        throw new RuntimeException("Failed to start service " + name
                + ": onStart threw an exception", ex);
    }
    return service;
}

```

看一下Installer的onStart方法：

很简单就是执行了mInstaller的waitForConnection方法，这里简单介绍一下Installer类，该类是系统安装apk时的一个服务类，继承SystemService（系统服务的一个抽象接口），需要在启动完成Installer服务之后才能启动其他的系统服务。

```
@Overridepublic void onStart() {
    Slog.i(TAG, "Waiting for installd to be ready.");
    mInstaller.waitForConnection();
}

```

然后查看waitForConnection（）方法：

通过追踪代码可以发现，其在不断的通过ping命令连接Zygote进程（SystemServer和Zygote进程通过socket方式通讯，其他进程通过Binder方式通讯）

```
public void waitForConnection() {
    for (;;) {
        if (execute("ping") >= 0) {
            return;
        }
        Slog.w(TAG, "installd not ready");
        SystemClock.sleep(1000);
    }
}

```

继续看startBootstrapServices方法：

这段代码主要是用于启动ActivityManagerService服务，并为其设置SysServiceManager和Installer。ActivityManagerService是系统中一个非常重要的服务，Activity，service，Broadcast，contentProvider都需要通过其余系统交互。

```
// Activity manager runs the show.mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();
mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
mActivityManagerService.setInstaller(installer);

```

首先看一下Lifecycle类的定义：

可以看到其实ActivityManagerService的一个静态内部类，在其构造方法中会创建一个ActivityManagerService，通过刚刚对Installer服务的分析我们知道，SystemServiceManager的startService方法会调用服务的onStart()方法，而在Lifecycle类的定义中我们看到其onStart（）方法直接调用了mService.start()方法，mService是Lifecycle类中对ActivityManagerService的引用

```
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context);
    }

    @Override
    public void onStart() {
        mService.start();
    }

    public ActivityManagerService getService() {
        return mService;
    }
}

```

4.3 启动部分服务

启动PowerManagerService服务：

启动方式跟上面的ActivityManagerService服务相似都会调用其构造方法和onStart方法，PowerManagerService主要用于计算系统中和Power相关的计算，然后决策系统应该如何反应。同时协调Power如何与系统其它模块的交互，比如没有用户活动时，屏幕变暗等等。 mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

然后是启动LightsService服务

主要是手机中关于闪光灯，LED等相关的服务；也是会调用LightsService的构造方法和onStart方法； mSystemServiceManager.startService(LightsService.class);

然后是启动DisplayManagerService服务

主要是手机显示方面的服务 mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

然后是启动PackageManagerService，该服务也是android系统中一个比较重要的服务

包括多apk文件的安装，解析，删除，卸载等等操作。

可以看到PackageManagerService服务的启动方式与其他服务的启动方式有一些区别，直接调用了PackageManagerService的静态main方法

```
Slog.i(TAG, "Package Manager");
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
    mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
mFirstBoot = mPackageManagerService.isFirstBoot();
mPackageManager = mSystemContext.getPackageManager();

```

看一下其main方法的具体实现：

可以看到也是直接使用new的方式创建了一个PackageManagerService对象，并在其构造方法中初始化相关变量，最后调用了ServiceManager.addService方法，主要是通过Binder机制与JNI层交互

```
public static PackageManagerService main(Context context, Installer installer, boolean factoryTest, boolean onlyCore) {
    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);
    ServiceManager.addService("package", m);
    return m;
}

```

然后查看startCoreServices方法：

可以看到这里启动了BatteryService（电池相关服务），UsageStatsService，WebViewUpdateService服务等。

```
private void startCoreServices() {
    // Tracks the battery level.  Requires LightService.
    mSystemServiceManager.startService(BatteryService.class);

    // Tracks application usage stats.
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));
    // Update after UsageStatsService is available, needed before performBootDexOpt.
    mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

    // Tracks whether the updatable WebView is in a ready state and watches for update installs.
    mSystemServiceManager.startService(WebViewUpdateService.class);
}

```

总结：

SystemServer进程是android中一个很重要的进程由Zygote进程启动；

```
SystemServer进程主要用于启动系统中的服务；

SystemServer进程启动服务的启动函数为main函数；

SystemServer在执行过程中首先会初始化一些系统变量，加载类库，创建Context对象，创建SystemServiceManager对象等之后才开始启动系统服务；

SystemServer进程将系统服务分为三类：boot服务，core服务和other服务，并逐步启动

SertemServer进程在尝试启动服务之前会首先尝试与Zygote建立socket通讯，只有通讯成功之后才会开始尝试启动服务；

创建的系统服务过程中主要通过SystemServiceManager对象来管理，通过调用服务对象的构造方法和onStart方法初始化服务的相关变量；

服务对象都有自己的异步消息对象，并运行在单独的线程中；

```

3\. Binder 机制（IPC、AIDL 的使用）
---------------------------

1、什么是AIDL以及如何使用（★★★★）

```
①aidl是Android interface definition Language 的英文缩写，意思Android 接口定义语言。 ②使用aidl可以帮助我们发布以及调用远程服务，实现跨进程通信。 ③将服务的aidl放到对应的src目录，工程的gen目录会生成相应的接口类 
```

我们通过bindService（Intent，ServiceConnect，int）方法绑定远程服务，在bindService中有一个ServiceConnec接口，我们需要覆写该类的onServiceConnected(ComponentName,IBinder)方法，这个方法的第二个参数IBinder对象其实就是已经在aidl中定义的接口，因此我们可以将IBinder对象强制转换为aidl中的接口类。

我们通过IBinder获取到的对象（也就是aidl文件生成的接口）其实是系统产生的代理对象，该代理对象既可以跟我们的进程通信，又可以跟远程进程通信，作为一个中间的角色实现了进程间通信。

2、AIDL的全称是什么?如何工作?能处理哪些类型的数据？（★★★）

AIDL全称Android Interface Definition Language（AndRoid接口描述语言） 是一种接口描述语言; 编译器可以通过aidl文件生成一段代码，通过预先定义的接口达到两个进程内部通信进程跨界对象访问的目的。

需要完成2件事情: 1\. 引入AIDL的相关类.; 2\. 调用aidl产生的class.理论上, 参数可以传递基本数据类型和String, 还有就是Bundle的派生类, 不过在Eclipse中,目前的ADT不支持Bundle做为参数。

3.android的IPC通信方式，线程（进程间）通信机制有哪些

```
1.ipc通信方式：binder、contentprovider、socket

2.操作系统进程通讯方式：共享内存、socket、管道

3.为什么使用 Parcelable，好处是什么？

```

实现机制 \* Interface for classes whose instances can be written to \* and restored from a {@link Parcel}. Classes implementing the Parcelable \* interface must also have a static field called `CREATOR`, which \* is an object implementing the {@link Parcelable.Creator Parcelable.Creator} \* interface. 如上，摘自Parcelable注释：如果想要写入Parcel或者从中恢复，则必须implements Parcelable并且必须有一个static field 而且名字必须是CREATOR.... 好吧，感觉好复杂。有如下疑问： 1、Parcelable是干啥的？为什么需要它？ 2、Parcel又是干啥的？ 3、如果是写入Parcel中、从Parcelable中恢复，那要Parcelable岂不是“多此一举”？

下面逐个回答上述问题：

1、Parcelable是干啥的？从源码看：

```
    public interface Parcelable {
        ...
        public void writeToParcel(Parcel dest, int flags);
        ...

```

简单来说，Parcelable是一个interface，有一个方法writeToParcel(Parcel dest, int flags)，该方法接收两个参数，其中第一个参数类型是Parcel。看起来Parcelable好像是对Parcelable的一种包装，从实际开发中，会在方法writeToParcel中调用Parcel的某些方法，完成将对象写入Parcelable的过程。

为什么往Parcel写入或恢复数据，需要继承Parcelable呢？我们看Intent的putExtra系列方法： 

往Intent中添加数据，无法就是添加以上各种类型，简单的数据类型有对应的方法，如putExtra(String, String)，复杂一点的有putExtra(String, Bundle),putExtra(String, Serializable)、putExtra(String, Bundle)、putExtra(String, Parcelable)、putExtra(String, Parcelable)。

现在想想，如果往Intent里添加一个我们自定义的类型对象person（Person类的实例），咋整？总不能用putExtra(String,person)吧？为啥，类型不符合啊！如果Person没有基础任何类，那它不可以用putExtra的任何一个方法，比较不存在putExtra(String, Object）这样一个方法。

那为了可以用putExtra方法，Person就需要继承一个可以用putExtra方法的类（接口），可以继承Parcelable——继承其他类（接口）也没有问题。

现在捋一捋：为了使用putExtra方法，需要继承Parcelable类——事实上，还有更深的含义，且看后面。

2、Parcel又是干啥的？前面说过，继承了Parcelable接口的类，如果不是抽象类，必须实现方法 writeToParcel，该方法有一个Parcel类型的参数，Parcel源码：

```
public final class Parcel {
...
    public static Parcel obtain() {
        final Parcel[] pool = sOwnedPool;
        synchronized (pool) {
            Parcel p;
            for (int i=0; i<POOL_SIZE; i++) {
                p = pool[i];
                if (p != null) {
                    pool[i] = null;
                    if (DEBUG_RECYCLE) {
                        p.mStack = new RuntimeException();
                    }
                    return p;
                }
            }
        }
        return new Parcel(0);
    }
...public final native void writeInt(int val);
public final native void writeLong(long val);

...

```

Parcel是一个final不可继承类，其代码很多，其中重要的一些部分是它有许多native的函数，在writeToParcel中调用的这些方法都直接或间接的调用native函数完成。

现在有一个问题，在public void writeToParcel(Parcel dest, int flags)中调用dest的函数，这个dest是传入进来的，是形参，那实参在哪里？没有看到有什么地方生成了一个Parcel的实例，然后调用writeToParcel啊？？那它又不可能凭空出来。现在回到Intent这边来，看看它的内部做了什么事：

```
    Intent i = new Intent();
    Person person = new Person();
    i.putExtra("person", person);
    i.setClass(this, SecondeActivity.class);
    startActivity(i);

```

为了简单说明情况，我写了如上的代码，就不解释了。看看putExtra做了什么事情，看源码：

```
public Intent putExtra(String name, Parcelable value) {
    if (mExtras == null) {
        mExtras = new Bundle();
    }
    mExtras.putParcelable(name, value);
    return this;
}

```

这里调用的putExtra的第二个参数是Parcelable类型的，也印证了前面必须要类型符合（这里多说一句，面向对象的六大原则里有一个非常非常重要的“里氏替换”原则，子类出现的地方可以用父类代替，这样所有继承了Parcelable的类都可以传入这个putExtra中）。原来这里用到了Bundle类，看源码：

```
public void putParcelable(String key, Parcelable value) {
    unparcel();
    mMap.put(key, value);
    mFdsKnown = false;
}

```

mMap是一个Map。看到这里，原来我们传入的person被写入了Map里面了。这个Bundle也是继承自Parcelable的。其他putExtra系列的方法都是调用这个mMap的put。为什么要用Bundle的类里的Map？统一管理啊！所有传到Intent的extra我都不管，交给Bundle类来管理了，这样Intent类就不会太笨重（面向对象六大原则之迪米特原则——我不管你怎么整，整对了就行）。看Bundle源码：

```
public final class Bundle implements Parcelable, Cloneable {
    private static final String LOG_TAG = "Bundle";
    public static final Bundle EMPTY;
  //Bundle类一加载就生成了一个Bundle实例
    static {
        EMPTY = new Bundle();
        EMPTY.mMap = Collections.unmodifiableMap(new HashMap<String, Object>());
    }    /\* package \*/ Map<String, Object> mMap = null;
    /\* package \*/ Parcel mParcelledData = null;

```

看到这里，还是没有发现Parcel实例在什么地方生成，继续往下看，看startActivity这个方法，找到最后会发现最终启动Activity的是一个ActivityManagerNative类，查看对应的方法：

```
public int startActivity(IApplicationThread caller, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, String profileFile, ParcelFileDescriptor profileFd, Bundle options) throws RemoteException {
    Parcel data = Parcel.obtain(); //在这里生成了Parcel实例
    Parcel reply = Parcel.obtain(); //又生成了一个Parcel实例
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    intent.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeStrongBinder(resultTo);
    data.writeString(resultWho);
    data.writeInt(requestCode);
    data.writeInt(startFlags);
    data.writeString(profileFile);
    if (profileFd != null) {
        data.writeInt(1);
        profileFd.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
    } else {
        data.writeInt(0);
    }
    if (options != null) {
        data.writeInt(1);
        options.writeToParcel(data, 0);
    } else {
        data.writeInt(0);
    }
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
    reply.readException();
    int result = reply.readInt();
    reply.recycle();
    data.recycle();
    return result;
}

```

千呼万唤的Parcel对象终于出现了，这里生成了俩Parcel对象：data和reply，主要的是data这个实例。obtain是一个static方法，用于从Parcel池（pool）中找出一个可用的Parcel，如果都不可用，则生成一个新的。每一个Parcel（Java）都与一个C++的Parcel对应。

```
public static Parcel obtain() {
    final Parcel[] pool = sOwnedPool;
    synchronized (pool) {
        Parcel p;
        for (int i=0; i<POOL_SIZE; i++) {
            p = pool[i];
            if (p != null) {
                pool[i] = null;
                if (DEBUG_RECYCLE) {
                    p.mStack = new RuntimeException();
                }
                return p;
            }
        }
    }
    return new Parcel(0);
}

```

在intent.writeToParcel(data, 0)里，查看源码：

```
public void writeToParcel(Parcel out, int flags) {
    out.writeString(mAction);
    Uri.writeToParcel(out, mData);
    out.writeString(mType);
    out.writeInt(mFlags);
    out.writeString(mPackage);
    ComponentName.writeToParcel(mComponent, out);

    if (mSourceBounds != null) {
        out.writeInt(1);
        mSourceBounds.writeToParcel(out, flags);
    } else {
        out.writeInt(0);
    }

    if (mCategories != null) {
        out.writeInt(mCategories.size());
        for (String category : mCategories) {
            out.writeString(category);
        }
    } else {
        out.writeInt(0);
    }

    if (mSelector != null) {
        out.writeInt(1);
        mSelector.writeToParcel(out, flags);
    } else {
        out.writeInt(0);
    }

    if (mClipData != null) {
        out.writeInt(1);
        mClipData.writeToParcel(out, flags);
    } else {
        out.writeInt(0);
    }

    out.writeBundle(mExtras); //终于把我们自定义的person实例送走了
}

```

看到这里Parcel实例终于生成了，但是我们重写的从Parcelable接口而来的writeToParcel这个方法在什么地方被调用呢？从上面的Intent中的out.writeBundle(mExtras)--\>writeBundle(Bundle val)--\>writeToParcel(Parcel parcel, int flags)--\>writeMapInternal(Map\<String,Object\> val)--\>writeValue(Object v)--\>writeParcelable(Parcelable p, int parcelableFlags)（除了out.writeBundle(mExtras)这个方法，其他的方法都是在Bundle和Parcel里面调来调去的，真心累！）:

```
public final void writeParcelable(Parcelable p, int parcelableFlags) {
    if (p == null) {
        writeString(null);
        return;
    }
    String name = p.getClass().getName();
    writeString(name);
    p.writeToParcel(this, parcelableFlags);//调用自己实现的方法
}

```

OK，终于出来了。。。到这里，写入的过程已经出来了。 那如何恢复呢？这里用到的是我们自己写的createFromParcel这个方法，从一个Intent中恢复person： Intent i= getIntent(); Person p = i.getParcelableExtra("person"); 调啊调，调到这个：

```
public final <T extends Parcelable> T readParcelable(ClassLoader loader) {
    String name = readString();
    if (name == null) {
        return null;
    }
    Parcelable.Creator<T> creator;
    synchronized (mCreators) {
        HashMap<String,Parcelable.Creator> map = mCreators.get(loader);
        if (map == null) {
            map = new HashMap<String,Parcelable.Creator>();
            mCreators.put(loader, map);
        }
        creator = map.get(name);
        if (creator == null) {
            try {
                Class c = loader == null ?
                    Class.forName(name) : Class.forName(name, true, loader);
                Field f = c.getField("CREATOR");
                creator = (Parcelable.Creator)f.get(null);
            }
            catch (IllegalAccessException e) {
                Log.e(TAG, "Class not found when unmarshalling: "
                                    + name + ", e: " + e);
                throw new BadParcelableException(
                        "IllegalAccessException when unmarshalling: " + name);
            }
            catch (ClassNotFoundException e) {
                Log.e(TAG, "Class not found when unmarshalling: "
                                    + name + ", e: " + e);
                throw new BadParcelableException(
                        "ClassNotFoundException when unmarshalling: " + name);
            }
            catch (ClassCastException e) {
                throw new BadParcelableException("Parcelable protocol requires a "
                                    + "Parcelable.Creator object called "
                                    + " CREATOR on class " + name);
            }
            catch (NoSuchFieldException e) {
                throw new BadParcelableException("Parcelable protocol requires a "
                                    + "Parcelable.Creator object called "
                                    + " CREATOR on class " + name);
            }
            if (creator == null) {
                throw new BadParcelableException("Parcelable protocol requires a "
                                    + "Parcelable.Creator object called "
                                    + " CREATOR on class " + name);
            }

            map.put(name, creator);
        }
    }

    if (creator instanceof Parcelable.ClassLoaderCreator<?>) {
        return ((Parcelable.ClassLoaderCreator<T>)creator).createFromParcel(this, loader);
    }
    return creator.createFromParcel(this); //调用我们自定义的那个方法
}

```

最后终于从我们自定义的方法中恢复那个保存的实例。

5.Android 图像显示相关流程，Vsync 信号等 Android Vsync 原理浅析 Preface Android中，Client测量和计算布局，SurfaceFlienger（server）用来渲染绘制界面，client和server的是通过匿名共享内存（SharedClient）通信。 每个应用和SurfaceFlienger之间都会创建一个SharedClient，一个SharedClient最多可以创建31个SharedBufferStack，每个surface对应一个SharedBufferStack，也就是一个Window。也就意味着，每个应用最多可以创建31个窗口。

Android 4.1 之后，AndroidOS 团队对Android Display进行了不断地进化和改变。引入了三个核心元素：Vsync，Triple Butter，Choreographer。 首先来理解一下，图形界面的绘制，大概是有CPU准备数据，然后通过驱动层把数据交给GPU来进行绘制。图形API不允许CPU和GPU直接通信，所以就有了图形驱动（Graphics Driver）来进行联系。Graphics Driver维护了一个序列（Display List），CPU不断把需要显示的数据放进去，GPU不断取出来进行显示。 其中Choreographer起调度的作用。统一绘制图像到Vsync的某个时间点。 Choreographer在收到Vsync信号时，调用用户设置的回调函数。函数的先后顺序如下： CALLBACK\_INPUT：与输入事件有关 CALLBACK\_ANIMATION：与动画有关 CALLBACK\_TRAVERSAL：与UI绘制有关 Vsync是什么呢？首先来说一下什么是FPS，FPS就是Frame Per Second（每秒的帧数）的缩写，我们知道，FPS\>=60时，我们就不会觉得动画卡顿。当FPS=60时是个什么概念呢？1000/60≈16.6，也就是说在大概16ms中，我们要进行一次屏幕的刷新绘制。Vsync是垂直同步的缩写。这里我们可以简单的理解成，这就是一个时间中断。例如，每16ms会有一个Vsync信号，那么系统在每次拿到Vsync信号时刷新屏幕，我们就不会觉得卡顿了。 但实现起来还是有点困难的。

多重缓冲是什么技术呢？我们先来说双重缓冲。在Linux上，通常使用FrameBuffer来做显示输出。双重缓冲会创建一个FrontBuffer和一个BackBuffer，顾名思义，FrontBuffer是当前显示的页面，BackBuffer是下一个要显示的画面。然后滚动电梯式显示数据。为什么呢？这样好在哪里呢？首先他并不是不卡了，他还是会卡。但是如果是单重缓冲，页面可能会有这种情况：A面数据需要显示，然后是B面数据显示，B面数据显示需要耗费一定时间，但是这个时间里，C面数据也请求了展示，我们可能会看到，在展示C面数据的时候，还有B面数据的残影… 下面分情况来具体说明一下（Vsync每16秒一次）。 1.没有使用Vsync的情况

可以看出，在第一个16ms之内，一切正常。然而在第二个16ms之内，几乎是在时间段的最后CPU才计算出了数据，交给了Graphics Driver,导致GPU也是在第二段的末尾时间才进行了绘制，整个动作延后到了第三段内。从而影响了下一个画面的绘制。这时会出现Jank（闪烁，可以理解为卡顿或者停顿）。那么在第二个16ms前半段的时间CPU和GPU干什么了？哦，他们可能忙别的事情了。这就是卡顿出现的原因和情况。CPU和GPU很随意，爱什么时候刷新什么时候刷新，很随意。 2.有Vsync的情况

如果，按照之前的前提来说，Vsync每16ms一次，那么在每次发出Vsync命令时，CPU都会进行刷新的操作。也就是在每个16ms的第一时间，CPU就会想赢Vsync的命令，来进行数据刷新的动作。CPU和GPU的刷新时间，和Display的FPS是一致的。因为只有到发出Vsync命令的时候，CPU和GPU才会进行刷新或显示的动作。图中是正常情况。那么不正常情况是怎么个情况？我们先来说一下双重缓冲，然后再说。 3.双重缓冲 逻辑就是和之前一样。多重缓冲页面在Back Buffer，然后根据需求来显示不同数据。但是会有什么问题呢（这就是2中提到的问题）？

首先我们看Display行，A页面需要了两个时间单位，为什么？因为B Buffer在处理的时候太耗时了。然后导致了，在第一个Vsync发出的时候，还在GPU还在绘制B Buffer。那么，刚好，第一个Vsync发出之后很短的时间，A页面展示完了，B Buffer的也在一开始的时候就不进行计算了。那么接下来的时间呢？屏幕还是展示着B Buffer，这时候就会造成Jank现象。他不会动，因为他在等下一个Vsync过来的时候，才会显示下一个数据。 那么，如图，在A Buffer过来的时候，展示B页面的数据。这个时候！重复了上一个情况，也是太耗时了，然后又覆盖了下一个Vysnc发 出的时间，再次造成卡顿！依次类推，会造成多次卡顿。这个时候就有了三重缓冲的概念。 4.三重缓冲

首先看图。我们看到，B Buffer依旧很耗时，同样覆盖了第一个Vsync发出的时间点。但是，在第一个Vsync发出的时候，C Buffer站了出来，说，我来展示这个页面，你去缓冲A后面需要缓冲的页面吧！然后会发生什么？然后就是出现了一个Jank…，但是这个Jank只在这一个时间单位出现，是可以忽略不计的。因为之后的逻辑都是顺畅的了。依次类推，除了A和B 两个图层在交替显示，还有个“第三者”在不断帮他们两个可能需要展示的数据进行缓冲。但是注意了： 只有在需要时，才会进行三重缓冲。正常情况下，只使用二级缓冲！ 另外，缓冲区不是越多越好。上图，C页面在第四个时间段才展示出来，就是因为中间多了一个Buffer（C Buffer）来进行缓冲。 但是，虽然谷歌给了你这么牛逼的前提逻辑，实际开发中你写的APP还是会卡，为什么呢？原因大概有两点： 1.界面太复杂。 2.主线程(UI线程)太忙。他可能还在处理用户交互或者其他事情。