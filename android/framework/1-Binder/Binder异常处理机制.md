# 概述

```
文中代码分析基于Android 10.0 (Q)
```

两个进程之间若是要进行Binder通信，那么发起通信的一端我们就称它为Client进程。Client进程调用每一个代理对象的方法，本质上都是一次跨进程通信。如果这个方法是同步方法（非oneway修饰），那么此调用过程将会经历如下几个阶段。

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210806101909.awebp)

对应用工程师而言，他只会看到浮在海面的冰山一角，至于隐藏在海面下的系统调用和跨进程通信，是无需他感知的。但无需感知并不代表不存在。这些中间过程若是发生了异常，终归是需要被处理的。

本文将重点阐述两个问题：

1. 如果Server进程中Binder实体对象的方法发生异常，该异常将会去向何处？
2. RemoteException的本质和类别。

## 1如果Binder实体对象的方法中发生异常，该异常将会去向何处？

### 1.1 Server端有什么影响？

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210806101924.awebp)

AIDL工具生成的Stub抽象类主要用于方法派发。因此实体方法中如果报出异常的话，异常将首先会报给Stub类的onTransact方法。以如下`intMethod`方法为例，如果其内部发生异常，则该异常将会被onTransact方法感知。

[out/soong/.intermediates/frameworks/base/core/tests/coretests/FrameworksCoreTests/android_common/xref/srcjars.xref/frameworks/base/core/tests/coretests/src/android/os/IAidlTest.java](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fandroid-10.0.0_r30%3Aout%2Fsoong%2F.intermediates%2Fframeworks%2Fbase%2Fcore%2Ftests%2Fcoretests%2FFrameworksCoreTests%2Fandroid_common%2Fxref%2Fsrcjars.xref%2Fframeworks%2Fbase%2Fcore%2Ftests%2Fcoretests%2Fsrc%2Fandroid%2Fos%2FIAidlTest.java%3Bl%3D106%3Bbpv%3D1%3Bbpt%3D1)

```java
106 @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
107 {
108   java.lang.String descriptor = DESCRIPTOR;
109   switch (code)
110   {
111     case INTERFACE_TRANSACTION:
112     {
113       reply.writeString(descriptor);
114       return true;
115     }
116     case TRANSACTION_intMethod:
117     {
118       data.enforceInterface(descriptor);
119       int _arg0;
120       _arg0 = data.readInt();
121       int _result = this.intMethod(_arg0);
122       reply.writeNoException();
123       reply.writeInt(_result);
124       return true;
125     }
复制代码
```

121行是调用intMethod的位置，其内部发生的异常并没有在onTransact中被处理，因此会继续上报给Binder.execTransactInternal方法。

[/frameworks/base/core/java/android/os/Binder.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fos%2FBinder.java%231000)

```java
1000      private boolean execTransactInternal(int code, long dataObj, long replyObj, int flags,
1001              int callingUid) {
1002          // Make sure the observer won't change while processing a transaction.
1003          final BinderInternal.Observer observer = sObserver;
1004          final CallSession callSession =
1005                  observer != null ? observer.callStarted(this, code, UNSET_WORKSOURCE) : null;
1006          Parcel data = Parcel.obtain(dataObj);
1007          Parcel reply = Parcel.obtain(replyObj);
1008          // theoretically, we should call transact, which will call onTransact,
1009          // but all that does is rewind it, and we just got these from an IPC,
1010          // so we'll just call it directly.
1011          boolean res;
1012          // Log any exceptions as warnings, don't silently suppress them.
1013          // If the call was FLAG_ONEWAY then these exceptions disappear into the ether.
1014          final boolean tracingEnabled = Binder.isTracingEnabled();
1015          try {
1016              if (tracingEnabled) {
1017                  final String transactionName = getTransactionName(code);
1018                  Trace.traceBegin(Trace.TRACE_TAG_ALWAYS, getClass().getName() + ":"
1019                          + (transactionName != null ? transactionName : code));
1020              }
1021              res = onTransact(code, data, reply, flags);
1022          } catch (RemoteException|RuntimeException e) {
1023              if (observer != null) {
1024                  observer.callThrewException(callSession, e);
1025              }
1026              if (LOG_RUNTIME_EXCEPTION) {
1027                  Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
1028              }
1029              if ((flags & FLAG_ONEWAY) != 0) {
1030                  if (e instanceof RemoteException) {
1031                      Log.w(TAG, "Binder call failed.", e);
1032                  } else {
1033                      Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
1034                  }
1035              } else {
1036                  // Clear the parcel before writing the exception
1037                  reply.setDataSize(0);
1038                  reply.setDataPosition(0);
1039                  reply.writeException(e);
1040              }
1041              res = true;
1042          } finally {
1043              if (tracingEnabled) {
1044                  Trace.traceEnd(Trace.TRACE_TAG_ALWAYS);
1045              }
1046              if (observer != null) {
1047                  // The parcel RPC headers have been called during onTransact so we can now access
1048                  // the worksource uid from the parcel.
1049                  final int workSourceUid = sWorkSourceProvider.resolveWorkSourceUid(
1050                          data.readCallingWorkSourceUid());
1051                  observer.callEnded(callSession, data.dataSize(), reply.dataSize(), workSourceUid);
1052              }
1053          }
1054          checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
1055          reply.recycle();
1056          data.recycle();
1057  
1058          // Just in case -- we are done with the IPC, so there should be no more strict
1059          // mode violations that have gathered for this thread.  Either they have been
1060          // parceled and are now in transport off to the caller, or we are returning back
1061          // to the main transaction loop to wait for another incoming transaction.  Either
1062          // way, strict mode begone!
1063          StrictMode.clearGatheredViolations();
1064          return res;
1065      }
复制代码
```

1021行是调用onTransact的位置。如果异常没有在此方法中被处理，将会进一步抛出给execTransact，最终进入JNI方法：JavaBBinder::onTransact。

异常的传递关系正如本节开始的那幅图所示，其最终的处理分为了两种情况。

#### 1.1.1 由Binder.execTransactInternal来处理异常

| Exception                     | Code                     |
| ----------------------------- | ------------------------ |
| SecurityException             | EX_SECURITY              |
| BadParcelableException        | EX_BAD_PARCELABLE        |
| IllegalArgumentException      | EX_ILLEGAL_ARGUMENT      |
| NullPointerException          | EX_NULL_POINTER          |
| IllegalStateException         | EX_ILLEGAL_STATE         |
| NetworkOnMainThreadException  | EX_NETWORK_MAIN_THREAD   |
| UnsupportedOperationException | EX_UNSUPPORTED_OPERATION |
| ServiceSpecificException      | EX_SERVICE_SPECIFIC      |

[/frameworks/base/core/java/android/os/Parcel.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fos%2FParcel.java%231868)

```java
1868      public final void writeException(@NonNull Exception e) {
1869          int code = 0;
1870          if (e instanceof Parcelable
1871                  && (e.getClass().getClassLoader() == Parcelable.class.getClassLoader())) {
1872              // We only send Parcelable exceptions that are in the
1873              // BootClassLoader to ensure that the receiver can unpack them
1874              code = EX_PARCELABLE;
1875          } else if (e instanceof SecurityException) {
1876              code = EX_SECURITY;
1877          } else if (e instanceof BadParcelableException) {
1878              code = EX_BAD_PARCELABLE;
1879          } else if (e instanceof IllegalArgumentException) {
1880              code = EX_ILLEGAL_ARGUMENT;
1881          } else if (e instanceof NullPointerException) {
1882              code = EX_NULL_POINTER;
1883          } else if (e instanceof IllegalStateException) {
1884              code = EX_ILLEGAL_STATE;
1885          } else if (e instanceof NetworkOnMainThreadException) {
1886              code = EX_NETWORK_MAIN_THREAD;
1887          } else if (e instanceof UnsupportedOperationException) {
1888              code = EX_UNSUPPORTED_OPERATION;
1889          } else if (e instanceof ServiceSpecificException) {
1890              code = EX_SERVICE_SPECIFIC;
1891          }
1892          writeInt(code);
1893          StrictMode.clearGatheredViolations();
1894          if (code == 0) {
1895              if (e instanceof RuntimeException) {
1896                  throw (RuntimeException) e;
1897              }
1898              throw new RuntimeException(e);
1899          }
1900          writeString(e.getMessage());
复制代码
```

针对如上8种异常，Server进程会将异常的信息序列化写入Parcel对象，然后经由驱动发送给Client进程。

#### 1.1.2 由JavaBBinder::onTransact来处理异常

[/frameworks/base/core/jni/android_util_Binder.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fandroid_util_Binder.cpp%23355)

```c++
355      status_t onTransact(
356          uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0) override
357      {
358          JNIEnv* env = javavm_to_jnienv(mVM);
359  
360          ALOGV("onTransact() on %p calling object %p in env %p vm %p\n", this, mObject, env, mVM);
361  
362          IPCThreadState* thread_state = IPCThreadState::self();
363          const int32_t strict_policy_before = thread_state->getStrictModePolicy();
364  
365          //printf("Transact from %p to Java code sending: ", this);
366          //data.print();
367          //printf("\n");
368          jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
369              code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
370  
371          if (env->ExceptionCheck()) {
372              ScopedLocalRef<jthrowable> excep(env, env->ExceptionOccurred());
373              report_exception(env, excep.get(),
374                  "*** Uncaught remote exception!  "
375                  "(Exceptions are not yet supported across processes.)");
376              res = JNI_FALSE;
377          }
复制代码
```

除了1.1中所述的8中异常外，其余所有异常都交由JavaBBinder::onTransact来处理。

368行是调用Java层execTransact方法的位置。当该方法抛出异常时，371行的异常检测将为true，而其后的373行会将异常打印出来。需要注意的是，这些信息并不会被发送回Client进程。以下为示例，这些Log是在Server进程中输出的。

```
2020-04-15 21:54:00.454 1433-1453/com.hangl.androidemptypage:server E/JavaBinder: *** Uncaught remote exception!  (Exceptions are not yet supported across processes.)
    java.lang.RuntimeException: android.os.RemoteException: Test by Hangl
        at android.os.Parcel.writeException(Parcel.java:1898)
        at android.os.Binder.execTransactInternal(Binder.java:1039)
        at android.os.Binder.execTransact(Binder.java:994)
     Caused by: android.os.RemoteException: Test by Hangl
        at com.hangl.androidemptypage.ServerB$ServiceB.sendMsg(ServerB.java:24)
        at com.hangl.androidemptypage.IServiceB$Stub.onTransact(IServiceB.java:64)
        at android.os.Binder.execTransactInternal(Binder.java:1021)
        at android.os.Binder.execTransact(Binder.java:994) 
复制代码
```

从上述示例中可以看出，此次Server进程真正抛出的异常为RemoteException，而RuntimeException只是对RemoteException的一层封装。

[/frameworks/base/core/java/android/os/Parcel.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fos%2FParcel.java%231894)

```java
1894          if (code == 0) {
1895              if (e instanceof RuntimeException) {
1896                  throw (RuntimeException) e;
1897              }
1898              throw new RuntimeException(e);
1899          }
复制代码
```

回到Binder.execTransactInternal，对于RemoteException1898行会将其进行封装再次抛出，而对于非以上8种的RuntimeException，1896行也会将其再次抛出。

综合以上两种处理情况，可以推断所有Binder实体对象方法中发生的异常都会被处理。无非一种是将异常信息发送给对端进程，另一种是将异常信息在本进程输出。而这些处理都不会使Server进程退出。

仔细思考这样设计也是很合理的。作为Server进程，它在什么时候执行，该执行些什么都不由自己掌控，而是由Client进程控制。因此抛出异常本质上与Client进程相关，让一个Client进程的行为导致Server进程退出显然是不合理的。此外，Server进程可能关联着千百个Client，不能由于一个Client的错误行为而影响本可以正常获取服务的其他Client。

### 1.2 Client端有什么影响？

Client端受到的影响完全取决于Server端如何处理异常。上文中已经阐明，Server进程会分两种情况来处理异常：一种是将异常信息发送给Client进程，另一个种是将异常信息在本进程中输出。以下按照这两个情况分别讨论Client进程受到的影响。

#### 1.2.1 从Parcel对象中读回异常信息

[out/soong/.intermediates/frameworks/base/core/tests/coretests/FrameworksCoreTests/android_common/xref/srcjars.xref/frameworks/base/core/tests/coretests/src/android/os/IAidlTest.java](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fandroid-10.0.0_r30%3Aout%2Fsoong%2F.intermediates%2Fframeworks%2Fbase%2Fcore%2Ftests%2Fcoretests%2FFrameworksCoreTests%2Fandroid_common%2Fxref%2Fsrcjars.xref%2Fframeworks%2Fbase%2Fcore%2Ftests%2Fcoretests%2Fsrc%2Fandroid%2Fos%2FIAidlTest.java%3Bl%3D442%3Bbpv%3D0%3Bbpt%3D1)

```java
442   @Override public int intMethod(int a) throws android.os.RemoteException
443   {
444     android.os.Parcel _data = android.os.Parcel.obtain();
445     android.os.Parcel _reply = android.os.Parcel.obtain();
446     int _result;
447     try {
448       _data.writeInterfaceToken(DESCRIPTOR);
449       _data.writeInt(a);
450       boolean _status = mRemote.transact(Stub.TRANSACTION_intMethod, _data, _reply, 0);
451       if (!_status && getDefaultImpl() != null) {
452         return getDefaultImpl().intMethod(a);
453       }
454       _reply.readException();
455       _result = _reply.readInt();
456     }
457     finally {
458       _reply.recycle();
459       _data.recycle();
460     }
461     return _result;
462   }
复制代码
```

Server进程通过Parcel对象发送的异常信息最终在454行被读回。

[/frameworks/base/core/java/android/os/Parcel.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fos%2FParcel.java%231983)

```java
1983      public final void readException() {
1984          int code = readExceptionCode();
1985          if (code != 0) {
1986              String msg = readString();
1987              readException(code, msg);
1988          }
1989      }
复制代码
```

[/frameworks/base/core/java/android/os/Parcel.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fos%2FParcel.java%232033)

```java
2033      public final void readException(int code, String msg) {
2034          String remoteStackTrace = null;
2035          final int remoteStackPayloadSize = readInt();
2036          if (remoteStackPayloadSize > 0) {
2037              remoteStackTrace = readString();
2038          }
2039          Exception e = createException(code, msg);
2040          // Attach remote stack trace if availalble
2041          if (remoteStackTrace != null) {
2042              RemoteException cause = new RemoteException(
2043                      "Remote stack trace:\n" + remoteStackTrace, null, false, false);
2044              try {
2045                  Throwable rootCause = ExceptionUtils.getRootCause(e);
2046                  if (rootCause != null) {
2047                      rootCause.initCause(cause);
2048                  }
2049              } catch (RuntimeException ex) {
2050                  Log.e(TAG, "Cannot set cause " + cause + " for " + e, ex);
2051              }
2052          }
2053          SneakyThrow.sneakyThrow(e);
2054      }
复制代码
```

Client端根据异常code，msg和stack trace重新构建出Exception对象，并将其抛出。由于readException方法并没有用`throws`修饰，所以如果该异常是Checked Exception（譬如RemoteException，IOException）就不能够直接抛出，否则会产生编译错误。因此这里采用了SneakyThrow来进行规避。

结合Server端异常处理的第一种情况，可以知道Client端只会读到8种RuntimeException中的一种。由于RuntimeException属于Unchecked Exception，因此编译过程并不会去检查对它的处理。换句话说，程序员在调用代理对象的方式时虽然会用try catch代码块，但通常只会去catch RemoteException，而不会去catch RuntimeException（对于很多程序员而言，不强制等于不做）。这样一来，Parcel读回来的RuntimeException将会导致Client进程退出。以下为示例，注意这些Log是在Client进程中输出的。

```
Timestamp: 02-27 19:20:16.623    Process: com.google.android.dialer    PID: 9782    Thread: main
java.lang.NullPointerException: Attempt to get length of null array
  at android.os.Parcel.createException(Parcel.java:2077)
  at android.os.Parcel.readException(Parcel.java:2039)
  at android.os.Parcel.readException(Parcel.java:1987)
  at android.app.IActivityTaskManager$Stub$Proxy.activityPaused(IActivityTaskManager.java:4489)
  at android.app.servertransaction.PauseActivityItem.postExecute(PauseActivityItem.java:64)
  at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:177)
  at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
  at android.os.Handler.dispatchMessage(Handler.java:107)
  at android.os.Looper.loop(Looper.java:214)
  at android.app.ActivityThread.main(ActivityThread.java:7356)
  at java.lang.reflect.Method.invoke(Native Method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
Caused by: android.os.RemoteException: Remote stack trace:
  at android.util.ArraySet.add(ArraySet.java:422)
  at com.android.server.wm.AppWindowToken.setVisibility(AppWindowToken.java:579)
  at com.android.server.wm.ActivityRecord.setVisibility(ActivityRecord.java:1844)
  at com.android.server.wm.ActivityStackSupervisor.realStartActivityLocked(ActivityStackSupervisor.java:774)
  at com.android.server.wm.ActivityStackSupervisor.startSpecificActivityLocked(ActivityStackSupervisor.java:979)
复制代码
```

上述Log表示进程`com.google.android.dialer`出现错误并退出。但如果了解Binder的异常机制，就会知道问题的根源不在`com.google.android.dialer`进程，而在`system_server`进程。

NullPointerException并不是`dialer`进程发生的异常，而是它从Parcel对象中读取的异常。通过IActivityTaskManager$Stub$Proxy.activityPaused可知，此时`dialer`进程正在和`system_server`进程通信。因此该NullPointerException是`system_server`进程中抛出的异常。再结合Remote stack trace，可知此异常是在调用ArraySet.Add时出现的。

结合这些信息，接下来调试的方向就不应该局限在`dialer`中，而是着眼于`system_server`进程。否则就会犯了头疼医头，脚痛医脚的问题。

#### 1.2.2 从Parcel对象中读回Error信息

[/frameworks/base/core/jni/android_util_Binder.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fandroid_util_Binder.cpp%23368)

```c++
368          jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
369              code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
370  
371          if (env->ExceptionCheck()) {
372              ScopedLocalRef<jthrowable> excep(env, env->ExceptionOccurred());
373              report_exception(env, excep.get(),
374                  "*** Uncaught remote exception!  "
375                  "(Exceptions are not yet supported across processes.)");
376              res = JNI_FALSE;
377          }
......
402          return res != JNI_FALSE ? NO_ERROR : UNKNOWN_TRANSACTION;
403      }
复制代码
```

对于8种RuntimeException之外的其余异常，Server进程会将它们交给JavaBBinder::onTransact来处理。1.1.2中只说明了Server进程会在本进程中输出异常，但并未提及可能对Client进程产生的影响。

376行将res赋值为JNI_FALSE，因此JavaBBinder::onTransact最终返回UNKNOWN_TRANSACTION。

[/frameworks/native/libs/binder/Binder.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fnative%2Flibs%2Fbinder%2FBinder.cpp%23123)

```c++
123  status_t BBinder::transact(
124      uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
125  {
126      data.setDataPosition(0);
127  
128      status_t err = NO_ERROR;
129      switch (code) {
130          case PING_TRANSACTION:
131              reply->writeInt32(pingBinder());
132              break;
133          default:
134              err = onTransact(code, data, reply, flags);
135              break;
136      }
137  
138      if (reply != nullptr) {
139          reply->setDataPosition(0);
140      }
141  
142      return err;
143  }
复制代码
```

UNKNOWN_TRANSACTION最终会传入IPCThreadState::executeCommand中，并赋值给如下的error。

[/frameworks/native/libs/binder/IPCThreadState.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fnative%2Flibs%2Fbinder%2FIPCThreadState.cpp%231228)

```c++
1228              if ((tr.flags & TF_ONE_WAY) == 0) {
1229                  LOG_ONEWAY("Sending reply to %d!", mCallingPid);
1230                  if (error < NO_ERROR) reply.setError(error);
1231                  sendReply(reply, 0);
1232              } else {
1233                  LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
1234              }
复制代码
```

1230行会将该error写入Parcel对象中，并通过驱动传回Client进程。

[/frameworks/native/libs/binder/IPCThreadState.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fnative%2Flibs%2Fbinder%2FIPCThreadState.cpp%23821)

```c++
821  status_t IPCThreadState::sendReply(const Parcel& reply, uint32_t flags)
822  {
823      status_t err;
824      status_t statusBuffer;
825      err = writeTransactionData(BC_REPLY, flags, -1, 0, reply, &statusBuffer);
826      if (err < NO_ERROR) return err;
827  
828      return waitForResponse(nullptr, nullptr);
829  }
复制代码
```

[/frameworks/native/libs/binder/IPCThreadState.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fnative%2Flibs%2Fbinder%2FIPCThreadState.cpp%231025)

```c++
1025  status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
1026      int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
1027  {
1028      binder_transaction_data tr;
1029  
1030      tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
1031      tr.target.handle = handle;
1032      tr.code = code;
1033      tr.flags = binderFlags;
1034      tr.cookie = 0;
1035      tr.sender_pid = 0;
1036      tr.sender_euid = 0;
1037  
1038      const status_t err = data.errorCheck();
1039      if (err == NO_ERROR) {
1040          tr.data_size = data.ipcDataSize();
1041          tr.data.ptr.buffer = data.ipcData();
1042          tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
1043          tr.data.ptr.offsets = data.ipcObjects();
1044      } else if (statusBuffer) {
1045          tr.flags |= TF_STATUS_CODE;
1046          *statusBuffer = err;
1047          tr.data_size = sizeof(status_t);
1048          tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
1049          tr.offsets_size = 0;
1050          tr.data.ptr.offsets = 0;
1051      } else {
1052          return (mLastError = err);
1053      }
1054  
1055      mOut.writeInt32(cmd);
1056      mOut.write(&tr, sizeof(tr));
1057  
1058      return NO_ERROR;
1059  }
复制代码
```

sendReply会调用writeTransactionData，由于1038行data中读取的err是UNKNOWN_TRANSACTION，所以最终写入驱动的数据并不是data本身，而是data中的err值。

Client端接受数据后，会将该err从IPCThreadState::waitForResponse一直传回给android_os_BinderProxy_transact。

[/frameworks/base/core/jni/android_util_Binder.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fandroid_util_Binder.cpp%231319)

```c++
1319      status_t err = target->transact(code, *data, reply, flags);
1320      //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();
1321  
1322      if (kEnableBinderSample) {
1323          if (time_binder_calls) {
1324              conditionally_log_binder_call(start_millis, target, code);
1325          }
1326      }
1327  
1328      if (err == NO_ERROR) {
1329          return JNI_TRUE;
1330      } else if (err == UNKNOWN_TRANSACTION) {
1331          return JNI_FALSE;
1332      }
1333  
1334      signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data->dataSize());
1335      return JNI_FALSE;
1336  }
复制代码
```

1319行返回的err是UNKNOWN_TRANSACTION，因此最终会在1331行直接返回，而不会调用signalExceptionForError来抛出异常。

[out/soong/.intermediates/frameworks/base/core/tests/coretests/FrameworksCoreTests/android_common/xref/srcjars.xref/frameworks/base/core/tests/coretests/src/android/os/IAidlTest.java](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fandroid-10.0.0_r30%3Aout%2Fsoong%2F.intermediates%2Fframeworks%2Fbase%2Fcore%2Ftests%2Fcoretests%2FFrameworksCoreTests%2Fandroid_common%2Fxref%2Fsrcjars.xref%2Fframeworks%2Fbase%2Fcore%2Ftests%2Fcoretests%2Fsrc%2Fandroid%2Fos%2FIAidlTest.java%3Bl%3D442%3Bbpv%3D0%3Bbpt%3D1)

```java
442   @Override public int intMethod(int a) throws android.os.RemoteException
443   {
444     android.os.Parcel _data = android.os.Parcel.obtain();
445     android.os.Parcel _reply = android.os.Parcel.obtain();
446     int _result;
447     try {
448       _data.writeInterfaceToken(DESCRIPTOR);
449       _data.writeInt(a);
450       boolean _status = mRemote.transact(Stub.TRANSACTION_intMethod, _data, _reply, 0);
451       if (!_status && getDefaultImpl() != null) {
452         return getDefaultImpl().intMethod(a);
453       }
454       _reply.readException();
455       _result = _reply.readInt();
456     }
457     finally {
458       _reply.recycle();
459       _data.recycle();
460     }
461     return _result;
462   }
复制代码
```

返回的err表现为450行的_status，如果该代理有默认的实现，那么最终会调用默认实现的intMethod。否则将会调用455行的readInt来获取返回值。由于此时的_reply对象中并无数据，因此读回的int值为0。

那如果返回的数据不是基本类型，而是引用类型呢？以下面方法举例。

[out/soong/.intermediates/frameworks/base/core/tests/coretests/FrameworksCoreTests/android_common/xref/srcjars.xref/frameworks/base/core/tests/coretests/src/android/os/IAidlTest.java](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fandroid-10.0.0_r30%3Aout%2Fsoong%2F.intermediates%2Fframeworks%2Fbase%2Fcore%2Ftests%2Fcoretests%2FFrameworksCoreTests%2Fandroid_common%2Fxref%2Fsrcjars.xref%2Fframeworks%2Fbase%2Fcore%2Ftests%2Fcoretests%2Fsrc%2Fandroid%2Fos%2FIAidlTest.java%3Bl%3D482%3Bbpv%3D0%3Bbpt%3D1)

```java
482    if ((0!=_reply.readInt())) {
483      _result = android.os.AidlTest.TestParcelable.CREATOR.createFromParcel(_reply);
484    }
485    else {
486      _result = null;
487    }
复制代码
```

若是返回的数据为引用类型，由于_reply中无数据，因此返回值最终为null。

总结一下，对于Server进程中返回的其余异常（除8种RuntimeException以外），Client进程将不会感知到。最终代理对象方法的返回值将由其类型决定，基本类型返回0，引用类型返回null。

## 2RemoteException的本质和类别

RemoteException不继承于RuntimeException，因此它是一种Checked Exception。对于程序中抛出的Checked Exception，它必须按照如下两种方式中的一种来处理，否则会在编译时报错。

- 用try catch代码块来捕获异常。
- 将该方法用throws关键字修饰，告诉调用者们此方法可能会抛出该异常。

Checked Exception会对程序员的代码有强制要求。之所以这么做，是因为该类异常希望程序员可以提前预知并做好准备，它们本可以被处理，用不着让进程退出。

以下是Oracle官网的解释，表示对于那些进程可以从中恢复的异常，都应该把它声明为Checked Exception。像远程调用/网络请求/IO请求之类的异常，它们反映的多是数据无法获取，但并不意味着进程到了无法继续运行的地步，因此用Checked Exception最为合适。

> Generally speaking, do not throw a `RuntimeException` or create a subclass of `RuntimeException` simply because you don't want to be bothered with specifying the exceptions your methods can throw.
>
> Here's the bottom line guideline: If a client can reasonably be expected to recover from an exception, make it a checked exception. If a client cannot do anything to recover from the exception, make it an unchecked exception.

对于Android Q而言，RemoteException的子类有三个：

- DeadObjectException
- TransactionTooLargeException
- DeadSystemException，是DeadObjectException的子类

### 2.1 DeadObjectException

[/frameworks/base/core/java/android/os/DeadObjectException.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fos%2FDeadObjectException.java%2320)

```java
20  /**
21   * The object you are calling has died, because its hosting process
22   * no longer exists.
23   */
24  public class DeadObjectException extends RemoteException {
复制代码
```

DeadObjectException反映的是Server进程已经挂掉，而Client进程仍旧尝试通信的错误。这句话是注释的含义，但不是问题的本质。事实上DeadObjectException不仅仅有对端进程挂掉这一种情况可以触发，binder线程池耗尽也会触发该异常。

[/frameworks/base/core/jni/android_util_Binder.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fandroid_util_Binder.cpp%231319)

```c++
1319      status_t err = target->transact(code, *data, reply, flags);
1320      //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();
1321  
1322      if (kEnableBinderSample) {
1323          if (time_binder_calls) {
1324              conditionally_log_binder_call(start_millis, target, code);
1325          }
1326      }
1327  
1328      if (err == NO_ERROR) {
1329          return JNI_TRUE;
1330      } else if (err == UNKNOWN_TRANSACTION) {
1331          return JNI_FALSE;
1332      }
1333  
1334      signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data->dataSize());
1335      return JNI_FALSE;
1336  }
复制代码
```

Client端Java层接收到的异常有两处来源，一处是根据Parcel对象中异常信息构建出的RuntimeException，另一处就是如上1334行根据BpBinder::transact返回值构建出的各种异常。

[/frameworks/base/core/jni/android_util_Binder.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjni%2Fandroid_util_Binder.cpp%23741)

```c++
741  void signalExceptionForError(JNIEnv* env, jobject obj, status_t err,
742          bool canThrowRemoteException, int parcelSize)
743  {
744      switch (err) {
......
778          case DEAD_OBJECT:
779              // DeadObjectException is a checked exception, only throw from certain methods.
780              jniThrowException(env, canThrowRemoteException
781                      ? "android/os/DeadObjectException"
782                              : "java/lang/RuntimeException", NULL);
783              break;
784          case UNKNOWN_TRANSACTION:
785              jniThrowException(env, "java/lang/RuntimeException", "Unknown transaction code");
786              break;
787          case FAILED_TRANSACTION: {
788              ALOGE("!!! FAILED BINDER TRANSACTION !!!  (parcel size = %d)", parcelSize);
789              const char* exceptionToThrow;
790              char msg[128];
791              // TransactionTooLargeException is a checked exception, only throw from certain methods.
792              // FIXME: Transaction too large is the most common reason for FAILED_TRANSACTION
793              //        but it is not the only one.  The Binder driver can return BR_FAILED_REPLY
794              //        for other reasons also, such as if the transaction is malformed or
795              //        refers to an FD that has been closed.  We should change the driver
796              //        to enable us to distinguish these cases in the future.
797              if (canThrowRemoteException && parcelSize > 200*1024) {
798                  // bona fide large payload
799                  exceptionToThrow = "android/os/TransactionTooLargeException";
800                  snprintf(msg, sizeof(msg)-1, "data parcel size %d bytes", parcelSize);
801              } else {
802                  // Heuristic: a payload smaller than this threshold "shouldn't" be too
803                  // big, so it's probably some other, more subtle problem.  In practice
804                  // it seems to always mean that the remote process died while the binder
805                  // transaction was already in flight.
806                  exceptionToThrow = (canThrowRemoteException)
807                          ? "android/os/DeadObjectException"
808                          : "java/lang/RuntimeException";
809                  snprintf(msg, sizeof(msg)-1,
810                          "Transaction failed on small parcel; remote process probably died");
811              }
812              jniThrowException(env, exceptionToThrow, msg);
813          } break;
......
861      }
862  }
复制代码
```

由780和806行可知，Java层接收到的DeadObjectException有两处来历：

- BpBinder::transact返回值为DEAD_OBJECT。
- BpBinder::transact返回值为FAILED_TRANSACTION，但是本次传输的Parcel小于200K。

#### 2.1.1 在什么情况下返回DEAD_OBJECT？

[/frameworks/native/libs/binder/IPCThreadState.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fnative%2Flibs%2Fbinder%2FIPCThreadState.cpp%23854)

```c++
854          case BR_DEAD_REPLY:
855              err = DEAD_OBJECT;
856              goto finish;
复制代码
```

当Binder驱动返回给用户空间的cmd为BR_DEAD_REPLY时，JNI层的signalExceptionForError将会接收到DEAD_OBJECT。由此可知，DEAD_OBJECT真正出生的地方是在驱动里。

驱动中返回BR_DEAD_REPLY的地方有很多，本文不会一一列举。这里只展示其中典型的一例。

```c
2972 static struct binder_node *binder_get_node_refs_for_txn(
2973 		struct binder_node *node,
2974 		struct binder_proc **procp,
2975 		uint32_t *error)
2976 {
2977 	struct binder_node *target_node = NULL;
2978 
2979 	binder_node_inner_lock(node);
2980 	if (node->proc) {
2981 		target_node = node;
2982 		binder_inc_node_nilocked(node, 1, 0, NULL);
2983 		binder_inc_node_tmpref_ilocked(node);
2984 		node->proc->tmp_ref++;
2985 		*procp = node->proc;
2986 	} else
2987 		*error = BR_DEAD_REPLY;
2988 	binder_node_inner_unlock(node);
2989 
2990 	return target_node;
2991 }
复制代码
```

当node->proc为null时，表明该Binder实体所在的进程已经退出。因此2987行会返回BR_DEAD_REPLY。

#### 2.1.2 在什么情况下返回FAILED_TRANSACTION？

[/frameworks/native/libs/binder/IPCThreadState.cpp](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fnative%2Flibs%2Fbinder%2FIPCThreadState.cpp%23854)

```c++
858          case BR_FAILED_REPLY:
859              err = FAILED_TRANSACTION;
860              goto finish;
复制代码
```

当Binder驱动返回给用户空间的cmd为BR_FAILED_REPLY时，JNI层的signalExceptionForError将会接收到FAILED_TRANSACTION。由此可知，FAILED_TRANSACTION真正出生的地方也是在驱动里。

驱动中返回BR_FAILED_REPLY的地方有很多，本文不会一一列举。这里只展示其中典型的一例。

```c
3267 	t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
3268 		tr->offsets_size, extra_buffers_size,
3269 		!reply && (t->flags & TF_ONE_WAY));
3270 	if (IS_ERR(t->buffer)) {
3271 		/*
3272 		 * -ESRCH indicates VMA cleared. The target is dying.
3273 		 */
3274 		return_error_param = PTR_ERR(t->buffer);
3275 		return_error = return_error_param == -ESRCH ?
3276 			BR_DEAD_REPLY : BR_FAILED_REPLY;
3277 		return_error_line = __LINE__;
3278 		t->buffer = NULL;
3279 		goto err_binder_alloc_buf_failed;
3280 	}
复制代码
```

binder_alloc_new_buf用于分配此次通信所需的binder_buffer，其中可能会出现诸多错误。

```c
387 	if (is_async &&
388 	    alloc->free_async_space < size + sizeof(struct binder_buffer)) {
389 		binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
390 			     "%d: binder_alloc_buf size %zd failed, no async space left\n",
391 			      alloc->pid, size);
392 		return ERR_PTR(-ENOSPC);
393 	}
......
442 		binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
443 				   "allocated: %zd (num: %zd largest: %zd), free: %zd (num: %zd largest: %zd)\n",
444 				   total_alloc_size, allocated_buffers,
445 				   largest_alloc_size, total_free_size,
446 				   free_buffers, largest_free_size);
447 		return ERR_PTR(-ENOSPC);
复制代码
```

上述代码展现了binder buffer空间耗尽时返回的错误：-ENOSPC，此错误最终会以BR_FAILED_REPLY的方式返回用户空间。

当此次通信的数据量<200K时，最终往Java层抛出的异常为DeadObjectException。而这次异常只表示对端进程的binder buffer耗尽，并非表示对端进程退出了。

### 2.2 TransactionTooLargeException

当Binder驱动返回BR_FAILED_REPLY且此次传输的数据大于200K，则Java层会接收到TransactionTooLargeException的错误。

需要注意的是，Binder驱动中返回BR_FAILED_REPLY的地方有很多，找不到合适的binder_buffer来传输数据只是其中的一种。

### 2.3 DeadSystemException

[/frameworks/base/core/java/android/os/RemoteException.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-10.0.0_r2%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fos%2FRemoteException.java%2358)

```java
58     @UnsupportedAppUsage
59     public RuntimeException rethrowFromSystemServer() {
60         if (this instanceof DeadObjectException) {
61             throw new RuntimeException(new DeadSystemException());
62         } else {
63             throw new RuntimeException(this);
64         }
65     }
复制代码
```

当Client用catch代码块捕获RemoteException时，如果此次binder通信的对端为system_server，该方法便可以使用rethrowFromSystemServer重新抛出异常。

当原本的异常为DeadObjectException时，那么新抛出的异常就是封装过的DeadSystemException。



# 参考

https://juejin.cn/post/6844904130008334349