> 上一篇文章我们对 Android AIDL 的原理与工作流程做了讲解，现在我们想一下服务器并不希望任意一个客户端都能访问我的方法，所以我们需要对 AIDL 加上权限校验。接下来我在本文为大家介绍 AIDL 权限校验的两种方法。

## 权限声明 首先需要在服务端的 AndroidMenifest 文件中声明所需权限

```ini
<permission android:name="com.example.zs.ipcdemo.permission.ACCESS_BOOK_SERVIC" android:protectionLevel="normal" />
<uses-permission android:name="com.example.zs.ipcdemo.permission.ACCESS_BOOK_SERVIC" />
复制代码
```

客户端 AndroidManifest.xml 中添加权限：

```ini
<uses-permission android:name="com.example.zs.ipcdemo.permission.ACCESS_BOOK_SERVIC" />
复制代码
```

## 第一种方法
在 Service 中的 onBind 方法中处理代码如下：

```csharp
public IBinder onBind(Intent t) {
         // 远程调用的验证方式
        int check = checkCallingPermission("com.example.zs.ipcdemo.permission.ACCESS_BOOK_SERVIC"); 
        if (check == PackageManager.PERMISSION_DENIED) {
            // 权限校验失败
             return null;
        } 
        return mBinder;
    }
复制代码
```

**这种方式我在代码里面一直尝试不成功！！！**

## 第二种方法 
通过上一篇文章我们知道当客户端通过 Proxy 调用服务器方法时首先通过 onTransact() 方法解析外进程调用，然后调用本地方法再返还给客户端。我们在这里先做权限校验，如果权限校验不过同则返回 false 直接跳过此方法，具体代码如下：

```java
 private Binder mBinder = new IBookManager.Stub() {
        // 权限校验
        private boolean checkPermission(Context context, String permName, String pkgName) {
            PackageManager pm = context.getPackageManager();
            if (PackageManager.PERMISSION_GRANTED == pm.checkPermission(permName, pkgName)) {
                return true;
            } else {
                return false;
            }
        }
        
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            // 权限校验
            String packageName = null;
            String[] packages = getPackageManager().getPackagesForUid(getCallingUid());
            if (packages != null && packages.length > 0) {
                packageName = packages[0];
            }
            if (packageName == null) {
                return false;
            }
            boolean checkPermission = checkPermission(getApplication(),
                    "com.example.zs.ipcdemo.permission.ACCESS_BOOK_SERVIC", packageName);
            return checkPermission && super.onTransact(code, data, reply, flags);
        }

        @Override
        public List<Book> getBookList() {
            return mBooks;
        }

        @Override
        public void addBook(Book book) {
            if (book != null) {
                mBooks.add(book);
                Log.d(TAG, "book size:" + mBooks.size());
            }
        }

        @Override
        public void registerListener(IOnNewBookArrivedListener listener) {
            mRemoteCallbackList.register(listener);
            final int N = mRemoteCallbackList.beginBroadcast();
            mRemoteCallbackList.finishBroadcast();
            Log.d(TAG, "registerListener, current size:" + N);
        }

        @Override
        public void unregisterListener(IOnNewBookArrivedListener listener) {
            boolean success = mRemoteCallbackList.unregister(listener);
            if (success) {
                Log.d(TAG, "unregister success.");
            } else {
                Log.d(TAG, "not found, can not unregister.");
            }
            final int N = mRemoteCallbackList.beginBroadcast();
            mRemoteCallbackList.finishBroadcast();
            Log.d(TAG, "unregisterListener, current size:" + N);
        }
    };
复制代码
```

权限校验的代码已经更新了GitHub：[github.com/christian-z…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fchristian-zs%2FIPCDemo "https://github.com/christian-zs/IPCDemo")

# 参考
https://juejin.cn/post/6844904201659645960