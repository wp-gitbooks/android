# 线索
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210808155303.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202403112148310.png)

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210622140340.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210731223143.png)

# 前言
如果要做OOM治理，首先搞清楚什么原因会导致OOM，android直接导致OOM的类型主要可以分为以下几种类型

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202403112149005.png)

# 导致OOM的直接原因

## 1.堆溢出以及堆内存中没有足够的连续内存空间
堆溢出的时候，抛出的异常如下所示

```java
java.lang.OutOfMemoryError
Failed to allocate a 12192780 byte allocation with 4194304 free bytes and 6MB until OOM    
```

在堆内存中如果没有足够的连续内存空间，这种情况下会在上面抛出的异常的基础下再多增加一些异常信息"failed due to fragmentation (required continguous free “<< required_bytes << “ bytes for a new buffer where largest contiguous free ” << largest_continuous_free_pages << “ bytes)"，这是因为堆中内存碎片太多导致的

## 2.FD(文件描述符)数量溢出
当程序打开或者新建文件的时候，系统会返回一个文件描述符，它是一个索引值，指向该进程打开文件的记录表，例如当我们用输出流打开文件的时候，系统就会返回给我们一个FD，FD是可能会出现泄露的，例如当输入输出流没有关闭的时候，具体关于FD的信息可以参考[FD泄露问题总结](https://blog.csdn.net/ws6013480777777/article/details/84594116)，如果出现了FD溢出，会抛出如下异常

```java
E/art: ashmem_create_region failed for 'indirect ref table': Too many open files
 java.lang.OutOfMemoryError: Could not allocate JNI Env
 at java.lang.Thread.nativeCreate(Native Method)
 at java.lang.Thread.start(Thread.java:730)
复制代码
```

## 3.线程数量溢出
不同的机型，允许的最大线程数量是不一样的，在华为的部分机型上，这个上限被修改的很低（大约500），比较容易出现线程数溢出的问题，线程溢出的时候会抛出下面的异常

```java
pthread_create failed: clone failed: Out of memory
Throwing OutOfMemoryError "pthread_create (1040KB stack) failed: Out of memory"
java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Out of memory
at java.lang.Thread.nativeCreate(Native Method)
at java.lang.Thread.start(Thread.java:1078)
复制代码
```

## 4.虚拟内存不足
在新建线程的时候，底层需要创建JNIEnv对象，并且分配虚拟内存，如果虚拟内存耗尽，会导致创建线程失败，并且抛出OOM异常，如下

```java
pthread_create failed: couldn't allocate 1073152-bytes mapped space: Out of memory
Throwing OutOfMemoryError with VmSize  4191668 kB "pthread_create (1040KB stack) failed: Try again"
java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Try again
at java.lang.Thread.nativeCreate(Native Method)
at java.lang.Thread.start(Thread.java:753)
复制代码
```

# 导致OOM的间接原因

## 1内存泄露

### 分析

####  1线上OOM现状

结合bugly分析，我们可以看到不同的系统版本OOM有着不同的异常信息，安卓8.0以下的机型主要是因为堆内存溢出或者堆中没有连续的内存空间，**安卓8.0以上的机型主要是因为线程数量溢出或者虚拟内存不足，还有少部分是因为FD数量溢出**

####  2内存泄露现状

程序是存在一定的内存泄露的，内存泄露有些是自己的代码造成的，有些是系统底层造成的，还有少部分与手机Rom相关

#### 3具体分析

目前的OOM，**安卓8.0以下的机型主要是因为堆内存**溢出或者堆中没有连续的内存空间，安卓   8.0以上的主要是因为线程数量溢出或者虚拟内存不足，还有少部分是因为FD数量溢出，android8.0以下Bitmap占用的内存是放在堆中，8.0以上是放在Native中，而且Native的内存占用是没有限制的，所以android8.0后很少出现堆内存溢出
 但是，在android8.0以上的OOM主要是和创建线程相关的，那我们就需要知道创建一个线程的时候底层到底做了什么事情，创建线程的流程图如下:

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202403112149428.png)

首先先判断FD数量是否超过了最大限制，是的话抛出异常，否则创建FD，然后判断虚拟内存是否足够，是的话创建JNIEnv对象，分配虚拟内存，否则就抛出异常，然后再判断虚拟内存是否足够，不够就抛出异常，足够就再判断线程数量是否超出了最大限制，如果是的话抛出异常，否则创建线程，分配虚拟内存，这就是一个线程创建的整个流程，在创建线程的时候，系统需要创建FD，还需要创建JNIEnv，还需要分配虚拟内存，也就是说，其实创建一个线程的开销是非常大的，而在安卓8.0以上的机型中OOM基本与线程创建相关，要么是FD数量超标，要么是虚拟内存不足，要么是线程数超标，综上所述，在Android8.0以上，如果要治理OOM，就需要从线程下手，核心手段就是尽量减少线程的创建，那么如何减少线程的创建呢？首先就需要了解到在哪个页面，哪行代码，创建了哪个线程，具体方案是[线程优化](https://juejin.cn/post/6948388639105613837)

在安卓8.0以下OOM的原因主要是堆溢出或者堆中缺少连续的内存空间，**安卓8.0以下的机型Bitmap的内存是分配在堆中的，bitmap占用的内存是非常大的，有的图片内存占用可以达到20M以上**，这就占用了极大的内存，因此，针对android8.0以下的OOM，Bitmap优化是主要方向，具体优化方案是[Bitmap和本地图片资源优化](https://juejin.cn/post/6948393362114215973)

另外我们的程序还有一定的内存泄露，间接造成了OOM，对于内存泄露，只能是借助工具一个个分析解决

# OOM堆溢出治理
[[OOM治理-抖音]]
[[OOM-线上-监控]]


# FD泄露治理
[[1.1androidFD泄露问题总结]]


# 线程数量溢出
[[1.2线程优化]]
[[OOM-线程]]

[[扒一扒抖音是如何做线程优化的]]

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303270846022.png)


# 参考
https://juejin.cn/post/6948397601515372557

