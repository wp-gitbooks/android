

# 一、Android为什么设计一个匿名共享内存，共享内存不能满足需求吗？

首先我们来思考一下共享内存和Android匿名共享内存最大的区别，那就是**共享内存往往映射的是一个硬盘中真实存在的文件，而Android的匿名共享内存映射的一个虚拟文件**。这说明Android又想使用共享内存进行跨进程通信，又不想留下文件，同时也不想被其它的进程不小心打开了自己进程的文件，因此使用匿名共享内存的好处就是：

1. 不用担心共享内存映射的文件被其它进程打开导致数据异常。
2. 不会在硬盘中生成文件，使用匿名共享内存的方式主要是为了通信，而且通信是很频繁的，不希望因为通信而生成很多的文件，或者留下文件。



# 为什么叫匿名共享内存？明明通过`iotc`设置了名字的？

这个问题在我看来是我之前对匿名这个词有些误解，其实匿名并不是没有名字，而是无法通过这些明面上的信息找到实际的对象，就像马甲一样。匿名共享内存也正是如此，虽然我们设置了名字，但是另外的进程通过同样的名字创建匿名共享内存却并不指向同一个内存了（代码验证过），虽然名字相同，但是背后的人却已经换了。这同时也回答上个问题，为什么匿名共享内存不用担心被其它进程映射进行数据读写（除非经过自己的同意，也就是通过binder传递了文件描述符给另一个进程）。



# 概述

https://www.jianshu.com/p/d9bc9c668ba6



Android 的 **匿名共享内存(Ashmem)** 基于 Linux 的共享内存，都是在**临时文件系统(tmpfs)**上创建虚拟文件，再映射到不同的进程。它可以让多个进程操作同一块内存区域，并且除了物理内存限制，没有其他大小限制。相对于 Linux 的共享内存，Ashmem 对内存的管理更加精细化，并且添加了**互斥锁**。Java 层在使用时需要用到 MemoryFile，它封装了 native 代码。 Java 层使用匿名共享内存的4个点： 

1. 通过 MemoryFile 开辟内存空间，获得 FileDescriptor； 
2. 将 FileDescriptor 传递给其他进程； 
3. 往共享内存写入数据； 
4. 从共享内存读取数据。



# 使用

## 创建 MemoryFile 和 数据写入

```
/**
 * 需要写入到共享内存中的数据
 */
private val bytes = "落霞与孤鹜齐飞，秋水共长天一色。".toByteArray()


/**
 * 创建 MemoryFile 并返回 ParcelFileDescriptor
 */
private fun createMemoryFile(): ParcelFileDescriptor? {
    // 创建 MemoryFile 对象，1024 是最大占用内存的大小。
    val file = MemoryFile("TestAshmemFile", 1024)

    // 获取文件描述符，因为方法被标注为 @hide，只能反射获取
    val descriptor = invokeMethod("getFileDescriptor", file) as? FileDescriptor

    // 如果获取失败，返回
    if (descriptor == null) {
        Log.i("ZHP", "获取匿名共享内存的 FileDescriptor 失败")
        return null
    }

    // 往共享内存中写入数据
    file.writeBytes(bytes, 0, 0, bytes.size)

    // 因为要跨进程传递，需要序列化 FileDescriptor
    return ParcelFileDescriptor.dup(descriptor)
}


/**
 * 通过反射执行 obj.name() 方法
 */
private fun invokeMethod(name: String, obj: Any): Any? {
    val method = obj.javaClass.getDeclaredMethod(name)
    return method.invoke(obj)
}
```

##  将文件描述符传递到其他进程

这里选择用 Binder 传递 ParcelFileDescriptor。 我们定义一个 Code，用于 C/S 两端通信确定事件：

```text
/**
 * 两个进程在传递 FileDescriptor 时用到的 Code。
 */
const val MY_TRANSACT_CODE = 920511
```

再在需要的地方 bindService：

```kotlin
// 创建服务进程
val intent = Intent(this, MyService::class.java)
bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
```

bind 成功之后将 文件描述符 和 数据大小 序列化，然后通过 Binder 传递到 Service 进程：

```kotlin
private val serviceConnection = object: ServiceConnection {

    override fun onServiceConnected(name: ComponentName?, binder: IBinder?) {
        if (binder == null) {
            return
        }

        // 创建 MemoryFile，并拿到 ParcelFileDescriptor
        val descriptor = createMemoryFile() ?: return

        // 传递 FileDescriptor 和 共享内存中数据的大小
        val sendData = Parcel.obtain()
        sendData.writeParcelable(descriptor, 0)
        sendData.writeInt(bytes.size)

        // 保存对方进程的返回值
        val reply = Parcel.obtain()

        // 开始跨进程传递
        binder.transact(MY_TRANSACT_CODE, sendData, reply, 0)

        // 读取 Binder 执行的结果
        val msg = reply.readString()
        Log.i("ZHP", "Binder 执行结果是：「$msg」")
    }

    override fun onServiceDisconnected(name: ComponentName?) {}

}
```

两个进程的文件描述符指向同一个文件结构体，文件结构体指向了一片内存共享区域(ASMA)，使得两个文件描述符对应到同一片ASMA中。

## 4. 在其他进程接收 FileDescriptor 并读取数据

先定义一个 MyService 用于开启子进程：

```kotlin
class MyService : Service() {

    private val binder by lazy { MyBinder() }

    override fun onBind(intent: Intent) = binder
}
```

再实现具体的 MyBinder 类，主要包含3个步骤： 1. 从序列化数据中读取 FileDescriptor 和 共享内存中保存的数据大小; 2. 根据 FileDescriptor 创建 FileInputStream； 3. 读取共享内存中的数据。

```kotlin
/**
 * 这里不必使用 AIDL，继承 Binder 类 重写 onTransact 即可。
 */
class MyBinder: Binder() {

    /**
     * 文件描述符 和 数据大小 通过 data 传入。
     */
    override fun onTransact(code: Int, data: Parcel, reply: Parcel?, flags: Int): Boolean {
        val parent = super.onTransact(code, data, reply, flags)
        if (code != MY_TRANSACT_CODE && code != 931114) {
            return parent
        }

        // 读取 ParcelFileDescriptor 并转为 FileDescriptor
        val pfd = data.readParcelable<ParcelFileDescriptor>(javaClass.classLoader)
        if (pfd == null) {
            return parent
        }
        val descriptor = pfd.fileDescriptor

        // 读取共享内存中数据的大小
        val size = data.readInt()

        // 根据 FileDescriptor 创建 InputStream
        val input = FileInputStream(descriptor)

        // 从 共享内存 中读取字节，并转为文字
        val bytes = input.readBytes()
        val message = String(bytes, 0, size, Charsets.UTF_8)

        Log.i("ZHP", "读取到另外一个进程写入的字符串：「$message」")

        // 回复调用进程
        reply?.writeString("Server 端收到 FileDescriptor, 并且从共享内存中读到了：「$message」")

        return true
    }

}
```

这里拿到 FileDescriptor 后不仅可以读也能写入数据，还可以再创建一个 MemoryFile 对象。



# 原理

https://github.com/huanzhiyazi/articles/issues/27#ch1.3





![image-20210404212324207](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210404212324.png)



在Android系统中，提供了独特的匿名共享内存子系统Ashmem(Anonymous Shared Memory)，它以驱动程序的形式实现在内核空间中。它有两个特点，**一是能够辅助内存管理系统来有效地管理不再使用的内存块，二是它通过Binder进程间通信机制来实现进程间的内存共享**。

ashmem并像Binder是Android重新自己搞的一套东西，而是利用了**Linux的 tmpfs文件系统**。tmpfs是一种可以基于RAM或是SWAP的高速文件系统，然后可以拿它来实现不同进程间的内存共享。

大致思路和流程是：

- Proc A 通过 tmpfs 创建一块共享区域，得到这块区域的 fd（文件描述符）
- Proc A 在 fd 上 mmap 一片内存区域到本进程用于共享数据
- Proc A 通过某种方法把 fd 倒腾给 Proc B
- Proc B 在接到的 fd 上同样 mmap 相同的区域到本进程
- 然后 A、B 在 mmap 到本进程中的内存中读、写，对方都能看到了

其实核心点就是**创建一块共享区域，然后2个进程同时把这片区域 mmap 到本进程，然后读写就像本进程的内存一样。**这里要解释下第3步，为什么要倒腾 fd，因为在 linux 中 fd 只是对本进程是唯一的，在 Proc A 中打开一个文件得到一个 fd，但是把这个打开的 fd 直接放到 Proc B 中，Proc B 是无法直接使用的。但是文件是唯一的，就是说一个文件（file）可以被打开多次，每打开一次就有一个 fd（文件描述符），所以对于同一个文件来说，需要某种转化，把 Proc A 中的 fd 转化成 Proc B 中的 fd。这样 Proc B 才能通过 fd mmap 同样的共享内存文件。

使用场景：进程间大量数据传输



# 应用

日志收集
https://tech.meituan.com/2018/02/11/logan.html

https://www.cnblogs.com/NeilZhang/p/11482815.html

https://juejin.cn/post/6844904202091626503#heading-2





