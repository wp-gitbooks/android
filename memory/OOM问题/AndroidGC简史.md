# 线索

| -                    | Android 4.4   | Android 5~6 | Android 7  | Android 8~9  | Android 10   |
| -------------------- | ------------- | ----------- | ---------- | ------------ | ------------ |
| 回收算法             | CMS           | CMS         | CMS        | CC           | CC           |
| 内存分配机制         | Single-Thread | Per-Thread  | Per-thread | Bump Pointer | Bump Pointer |
| 内存分配性能         | 1x            | 4-5x        | 10x        | 18x          | 18x          |
| 临时变量开销         | 高            | 低(分代)    | 低(分代)   | 中           | 低(分代)     |
| 内存整理(防止碎片化) | 后台          | 后台/事件   | 后台/事件  | 前台并发     | 前台并发     |


从碎片化、单线程到多线程并发，后台到前台


[[#2 如果应用真的用了几个 GB 的内存，会发生什么？]]
kswapd
lmk


# Android GC 简史

Android 开发者对于 GC 既熟悉又陌生，听说过很多虎狼之词，对一些问题又不置可否；今天聊聊 Android 里的 GC，如果你对于下面的问题有兴趣又没答案，那你应该会有些收获：

> 1. JVM、Dalvik、ART, 它们之间是什么关系？
> 2. 所有版本的 Android 都是分代管理堆内存吗？
> 3. 垃圾对象到底是怎么被回收的？
> 4. 「内存抖动」你怕不怕？
> 5. 作为一个应用层开发者，我真的需要关心 Android GC 吗？

------

## 前言：概念辨析

为了避免一些朋友不是很清楚概念，在正文开始之前，先简单辨析一下：

**GC** GC，是指垃圾回收 (Garbage Collection)，是一些语言管理内存的方式，如 Java 语言等；程序员不需要主动管理内存，程序运行时环境(虚拟机)会做垃圾回收的工作，就是在**合适的时机 自动释放**不再需要的内存。

与 GC 对应的是 native 语言，它需要程序员主动释放申请内存，忘记释放或者释放时机不合适都会产生问题。

**GC Root** 在 native 语言中，内存的申请和释放需要程序员来操作，做到正确地申请和释放内存就是程序员要考虑的问题。

在类似 Java 这种使用了 GC 的语言中，程序员不关心内存的释放。正确地释放内存就是 GC 的责任，GC 的原则是保证正确性的前提下，尽可能提升性能。 于是就使用了 GC Root 的机制，逻辑上就是：GC 认为 GC Root 以及它引用的对象是程序后面的可能会用到的，所以不会释放；没有被 GC Root 直接或间接引用的对象，后面一定不会被用到，可以被释放掉。

在 Java 中常被用于GC Root的类型如下：

- (函数未出栈时的)局部变量
- 静态变量
- 存活状态的线程
- Native 方法中 JNI 引用的对象

另外，Java 中内存泄漏的本质，就是对象 (不当地) 被GC Root 引用导致无法释放。

之所以会在这里提到 GC Root, 因为后面的流程中上来就先找 GC Root 集合，看完这个你就知道那是在干嘛了。

**JVM / Dalvik / ART** JVM 是 Java 语言提出的虚拟机标准，基于这个标准各个厂商会有自己的实现。比如 Oracle 的 HotSpot，以及 Android 中的 DVM (Dalvik Virtual Machine) 和 ART (Android Runtime) 两个实现。其中，

- Dalvik 是 Android 4.4 (Kitkat) 以及之前的版本的虚拟机；
- ART 在 Android 4.4 (Kitkat) 引入，并在Android 5.0 (Lollipop) 开始取代Dalvik.

**分代管理** 分代管理是很多 JVM 虚拟机对于堆 (heap) 内存的管理机制，最为人熟知的是新生代 (Young Generation)、老年代 (Old Generation) 和永久代 (Permanent Generation) 这一组名词，这也是很多人讲 GC 时的默认组合。事实上，这一组名词是 Oracle 的 HotSpot 中对于 JDK 1.7及之前的实现方式。至于其他的 JVM 实现，可以选择是否采用分代管理的机制。

比如 **Dalvik 就没有采用分代管理的机制，ART 在 Android 8 (Oero) 和 9 (Pie) 版本未使用分代管理、其他的版本又采用了该机制**。

在分代管理的虚拟机中，新生代和老年代正如它们的名字一样，分别存储了新创建的对象和存活了很久的对象；新的对象会在新生代中创建，经过一定次数的 GC 后依然存活的对象，会被复制到老年代中(有些虚拟机新生代分为更多的区域，以达到的最佳的性能表现)。

------

前言结束，现在进入今天最重要的部分了

## Android GC 的演进

### 1史前时代的 Dalvik

Dalvik 虚拟机随着 Android 一起诞生，一直到 Android 4.4。

HTC G1 是第一款 Android 设备，内存为192MB，应用程序可用的堆内存仅为30MB左右，Dalvik 就是在这样的环境下诞生的。它的设计原则中，节省空间 (尽量让程序跑起来) 是第一优先级。

![HTC G1](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160105.image)

#### 内存分配

分配内存时，Dalvik 使用的算法是 `dlmalloc` (以 `java.util.concurrent` 作者 *Doug Lea* 命名的算法)，使用了单独的进程来分配内存，内存的分配效率较低。

- 在整个堆中寻找适合分配的内存 ![在整个堆中寻找适合分配的内存](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160117.image)

- 找到了适合分配的内存 ![找到了适合分配的内存](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160130.image)

- 内存分配成功 ![内存分配成功](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160135.image)

- 上面的图中给出了顺利分配内存的流程，如果当前堆中没有合适分配的内存，就会触发一个 `GC_FOR_ALLOC` 进入GC流程。

  ![没有合适分配的内存，就会触发一个  GC_FOR_ALLOC 进入GC流程](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160139.image)

#### 回收流程
- 标记GC Root 集合，这一步会导致应用暂停 ![标记GC Root 集合，这一步会导致应用暂停](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160145.image)
- 标记可触达对象1 ![标记可触达对象1](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160158.image)
- 标记可触达对象2，这一步会导致应用暂停 ![标记可触达对象2，这一步会导致应用暂停](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160205.image)
- 清理对象

![清理对象](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160210.image)

在 Dalvik 的一次回收过程中，有两个步骤会导致应用暂停(所有线程挂起)共10ms左右，这个时间还是比较长的，在一帧16.6ms的绘制时间里如果发生一次 GC， 很可能导致丢帧。

如果回收过后依然没有足够的空间，此时可能发生两件事：

- 增大堆体积
- (堆体积已经最大) 抛出 OutOfMemoryError

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160221.image)

**Dalvik 的问题：碎片化** 前面已经提到过，Dalvik 使用 `dlmalloc` 作为分配内存的算法，作者 *Doug Lea* 先生自己的文章 [A Memory Allocator](http://gee.cs.oswego.edu/dl/html/malloc.html) 中也提到了避免碎片化的问题，但Dalvik的内存碎片化问题依然严重。

- 比如Google I/O 中的例子：有 200MB 剩余空间 (200个 1MB 的内存碎片)，尝试分配 2MB 空间抛出了 OOM 错误。

![有 200MB 剩余空间 (200个 1MB 的内存碎片)，尝试分配 2MB 空间抛出了OOM错误。](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160228.image)

时代滚滚向前，Dalvik 在 Android 4.4之后被 ART取代，我们就把 Dalvik 的岁月称作史前时代吧~

### 2走向共和的 ART

ART 在 Android 4.4 引入(与 Dalvik 同时存在)，从 Android 5.0 开始成为唯一选择，并一直更新到现在(写作这篇文章的时间是2021年)。

#### Android 5.0 ~ 7.0

Android 5.0 ~ 7.0 (Nougat)， ART做了很多更新，但从 GC 的角度来看，可以放在一起讨论。

##### 引入分代管理

将堆分为新生代 (Young Generation) 和老年代 (Old Generation)，对应的GC也分为两种：

- Minor GC: 针对新生代的垃圾回收
- Major GC (Full GC) : 针对整个堆的垃圾回收

引入分代管理是基于这样的假设：生命周期短的对象 (如临时生成的对象) 的创建和销毁要比 (生命周期) 长的对象更加频繁。

程序运行时，触发的大部分是 Minor GC，只会针对新生代做处理，这样比 Dalvik 的全局回收要高效很多，极大降低了创建临时变量的开销。

##### 内存分配

与 Dalvik 不同， ART 中的内存分配使用了 `RosAlloc` 算法。做了以下的改进：

- 改为当前线程 (Thread-Local) 分配内存，与之对应的是 Dalvik 的单一线程负责内存分配。
- 降低了锁的颗粒度
- 针对小块内存的优化：归并处理
- 针对大块内存分配的优化

相比于 Android 4.4 的 Dalvik，Android 5.0 内存分配的性能提升到 4 - 5x，在Android 7.0 提升到了10x左右。

##### 内存回收
与 Dalvik 类似，Android 5.0 ~ 7.0 的内存回收也分为4个阶段(如下图)，其中第一个阶段(标记 GC Root 集合) 改为并发处理，使整个过程只需要暂停应用(挂起所有线程) 3ms 左右，提升明显。 ![ART 5.0~7.0 内存回收过程](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160246.image)

##### 整理内存

为了解决碎片化的问题，ART 会在应用后台时整理内存 (compaction)，就是把不连续的内存整理(使用copy)为连续的内存；这样空闲的内存中就有了更多的连续内存，避免了碎片化的问题，重复上面在 Kitkat的实验也不会抛出 OOM 了。

当然，当应用在前台时，内存整理也可能被触发；比如应用 GC 后仍然没有足够的连续内存。

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160300.image)

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160409.image)

#### Android 8.0 ~ 9.0

Android 8.0 开始，ART 引入了 Concurrent Copying (CC) GC，将整理内存的工作由后台转移到了前台，并且**移除了分代管理机制**。

##### 内存分配

从Android 8.0 开始，ART 引入了 `Bump Allocator` 的机制来分配内存，处理方式如下：

- 堆内存被分割为多个 region
- 每个线程对应一个 region, 并且维护一个指向下一个空闲空间的指针 `Bump Pointer`
- 分配内存时，直接 `Bump Pointer` 指针指向的地址分配即可，由于每个线程有对应的region，所以分配的过程是并发的，非常高效。
- 当前 region 剩余空间不够时，触发 compaction 操作，具体步骤见下一节。

基于上面的优化，相比 Android 4.4 的 Dalvik，ART 在 Android 8.0 的内存分配性能提升到18x，表现已经好于大部分的带锁对象池实现了。所以在使用对象池机制之前，先测试一下性能对比吧。

##### 内存回收

在 Concurrent Copying (CC) GC 中，GC时会遍历每个region，根据当前的对象状态来决定是否进行 copy 操作，具体过程如下：

- 找到 GC Root 并标记 GC Root 引用的对象 ![找到 GC Root 并标记 GC Root 引用的对象](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160321.image)
- 标记出未被 GC Root 引用的对象(垃圾对象) ![标记出未被 GC Root 引用的对象(垃圾对象)](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160420.image)
- 对region进行分析，决定每一个region是否进行copy ![对region进行分析，决定每一个region是否进行copy](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160443.image)

> 经过分析，垃圾对象较多的区域会被搬移， 而垃圾对象较少的区域不会被搬移，原因这个区域大部分对象后面还会用到，copy操作是有成本的，全量copy不划算

- copy 对象到新的 region ![copy 对象到新的 region](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160426.image)
- 清空搬移后的region，完成回收 ![清空搬移后的region，完成回收](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160456.image)

#### Android 10 及以后

从 Android 10 开始，Concurrent Copying (CC) GC 中加入了**分代管理机制**(也不知道当时为啥要移除)，有了分代管理，对应的 Minor GC 与 Major GC 也就回来，这一块我们重点讲一下对应的工作流程：

在 Generational CC 中，堆内存并没有显式地划分为不同的代，而是在运行时 把不同的 region 标记为新生代或者老年代；

##### CC Minor GC

- 根据 GC Root，标记新生代 region 的对象 ![标记新生代 region 的对象](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160504.image)
- Minor GC 不会导致追踪(trace)老年代region中的对象，但如果新生代region中的对象被老年代region引用，还是要在copy后更新对应的引用，这里用到了一个Remember Set的机制，将这一操作的开销讲到最小。 ![Minor GC 不会导致追踪(trace)老年代region中的对象](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160512.image)
- 新生代对象 copy 到新的 region 中，原region被回收，Minor GC 完成 ![新生代对象 copy 到新的 region 中，原region被回收，Minor GC 完成](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160518.image)

##### CC Major GC:

- 追踪所有的region，(根据规则)标记要回收的 region ![追踪所有的region，(根据规则)标记要回收的 region](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160526.image)
- 将需要回收的 region 中的对象 copy 到新的 region 中，回收原来的 region

![将需要回收的 region 中的对象 copy 到新的 region 中，回收原来的 region](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160534.image)

根据上面的流程，我们可以看到在未加入分代回收之前，Concurrent Copying GC 中每一次GC，都是一次 Major GC，这样回收性能得到提升，尤其是对于生命周期较短的对象。

### Summary 性能总结

各个版本性能对比图

| -                    | Android 4.4   | Android 5~6 | Android 7  | Android 8~9  | Android 10   |
| -------------------- | ------------- | ----------- | ---------- | ------------ | ------------ |
| 回收算法             | CMS           | CMS         | CMS        | CC           | CC           |
| 内存分配机制         | Single-Thread | Per-Thread  | Per-thread | Bump Pointer | Bump Pointer |
| 内存分配性能         | 1x            | 4-5x        | 10x        | 18x          | 18x          |
| 临时变量开销         | 高            | 低(分代)    | 低(分代)   | 中           | 低(分代)     |
| 内存整理(防止碎片化) | 后台          | 后台/事件   | 后台/事件  | 前台并发     | 前台并发     |

> Tip:
>
> CMS: Concurrent Mark Sweap CC: Concurrent Copying

------

## 另外几个问题

#### 0 GC 的日志要怎么看啊？

Dalvik 与 ART 的 GC 日志不太一样，具体差异可以参考[这里](https://developer.android.com/studio/debug/am-logcat#memory-logs)，有一点需要注意: ART 已经不会打印全部的GC日志了，那样就太频繁了。它只会打印以下三种：

- 程序主动请求的GC
- pause 时间超过5ms
- 总时间超过100ms

------

#### 1 Android 对于 Bitmap 内存处理的演进

这里还是要提一下 Bitmap，作为 Android 中最常见的“大对象”，它与 GC 的关系也很密切；Bitmap 最占内存的是它的像素数据，对于像素数据的储存，Android 发生过一些变化。

**Android 3.0 之前**

Bitmap像素数据存储在 native 堆中，数据的释放依赖 Java对象的 `finalize()` 方法回调，该方法的调用不太可靠，而且现在已经被 Java 标记为废弃。

**Android 3.0 ~ 7.0**

Bitmap 像素数据存储在 Java 堆中，确实解决了可靠释放的问题；也带来一个新问题：Android 应用程序对 Java 堆的限制是很严格的，一般不超过512MB (因厂家而已)，创建Bitmap这种大的对象很容易引起 GC，另外，如果大家经历过相对早期的Android开发，一定很熟悉部分设备上创建 Bitmap 导致 OOM 的问题。

**Android 8.0 及以后**

Bitmap 像素数据存储在 native 堆中，同时引入了 `NativeAllocationRegistry` 机制保证了 native 内存释放的可靠性，同时可以用的空间大大增加。

程序员最烦这种来回改的需求( native ==> Java ==> native )，但这里并不是没事找事，而是为了解决当时面临的问题。时至今日，Bitmap 依然可能造成内存使用的错误，即使 native 的最大可用空间为几个 GB，但也不能毫无顾忌地使用，一方面，native 内存压力大时也会触发 Java 内存的 GC (想不到吧)；另一个原因留给下一个问题。

------

#### 2 如果应用真的用了几个 GB 的内存，会发生什么？

正如上一个问题所述，native 内存的最大可用空间可以为几个 GB，如果真的用了这么多内存，也不释放。会发生什么呢？

我们上面讨论的所有 GC 问题都是关于当前应用的内存使用，它也会影响系统整体的内存使用情况。系统也有对应的机制来统筹系统资源的使用：kswapd 和 lmk

**kswapd**

kernel swap daemon，作为一个守护进程会一直监控系统内存的使用，**剩余内存达到低点(阈值)时触发回收操作**，剩余内存达到高点(阈值)时停止回收操作。回收策略：

1. 删除缓存的内存(缓存本是用来以空间换时间的，现在空间不足了，就释放掉)
2. 压缩内存中的数据(这些数据删除就丢失了，于是压缩后放在内存中的特定区域，节省了空间)

**lmk**

low memory killer, kswapd处理后内存依然不够用的话，就会触发 lmk 机制，简单讲就是**按照优先级(一个分数)从低到高，有序地关闭进程来释放资源。**

在回收的各个阶段，应用可以实现 `onTrimMemory(level)` 方法收到回调，可以根据对应 `level` 主动释放一些缓存数据，进程的优先级可以查看下面这张图:

![5851621951536_.pic_hd.jpg](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160602.image)

这里只是简单讲述逻辑，关于 kwapd 和 lmk 的更多细节可以参考[官方文档](https://developer.android.com/topic/performance/memory-management)

回到当前问题，应用用了几个 GB 的内存后的表现，会设备相关，可能会继续流畅运行，也可能进程被关闭。

------

#### 3 “内存抖动” 你怕不怕？

“内存抖动”可能是被面试官带火的一个词，实际的原理是比较简单的：

如果高频地申请较大尺寸的内存，则可能导致短时间内频繁触发 GC，造成内存的频繁申请和释放，使用Profiler查看内存使用时，看起来就是一个抖动的曲线。 ![看，内存“抖动”起来了~](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160609.image)

在 ART 之后，单次 GC 导致应用暂停时间不会很长，但如果连续触发多次 GC 就可能导致一帧 UI 绘制的延迟，造成丢帧。大家都知道不要在 `onDraw()` 里创建对象；我从来没有在 `onDraw()` 里创建大的对象，事实上在编码的时候如果有关注性能，写出“内存抖动”的代码是不容易的；但为了验证这个问题，我测试了各个Android “内存抖动”场景的表现，具体参数就请查看[针对「内存抖动」的一次测试](https://www.jianshu.com/p/6239672cf0d5)

直接说结论吧: Dalvik上遭遇内存抖动的代价是致命的，ART 上各个版本的表现差异不大，都会在1秒内有几次绘制延时，我们看到 ART 的巨大进步，同时也必须要避免写出这样的代码。。。

------

### 写在最后：

**作为应用层开发者，我真的需要关心GC吗？**

写这篇文章的时间是2021年，随着时间的推移，Dalvik 设备的占比已经很小了，小到10亿日活的微信也不再支持。 ![微信不再支持Lollipop以下的版本](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808160622.image)

伴随着 ART 的推出和迭代，它变得很好了，我们能对 GC 做的优化越来越少了，我们不需要格外关心GC，关心业务，关心应用的性能表现就足够了。

Android已经迭代了十几年，相比于之前的大刀阔斧，现在的重点在于细节上的深耕；随着版本的更替，有些问题被时间解决了；当然，我们针对 Dalvik 做过的所有[优化](https://juejin.cn/post/6844903705028853767)，也随之尘封到不为人知的历史中，烟消云散；

Android 团队的所有努力，就是让应用层的程序员只关心应用层的业务。从 GC 的角度，他们基本做到了这一点。

------

能看到这里的朋友，建议阅读/观看下面的参考资料，价值连城~

**Reference**

[Manage your app's memory](https://developer.android.com/topic/performance/memory)

[Memory allocation among processes](https://developer.android.com/topic/performance/memory-management)

[Trash talk (Android Dev Summit '18)](https://www.youtube.com/watch?v=Zc4JP8kNGmQ)

[Android memory and games (Google I/O'19)](https://www.youtube.com/watch?v=Do7oYWwOXTk&list=LL&index=2&t=983s)

[Deep dive into the ART runtime (Android Dev Summit '18)](https://www.youtube.com/watch?v=vU7Rhcl9x5o&list=LL&index=4&t=1721s)

[Understanding Android Runtime (ART) for faster apps (Google I/O'19)](https://www.youtube.com/watch?v=1uLzSXWWfDg)

[Episode 79: Picking Up Garbage (Android Developers Backstage)](https://podcasts.google.com/feed/aHR0cHM6Ly9hZGJhY2tzdGFnZS5saWJzeW4uY29tL3Jzcw==/episode/dGFnOmJsb2dnZXIuY29tLDE5OTk6YmxvZy0xMDUyMTA4NTQ3OTk4MDgyNDY1LnBvc3QtMzQ0MTc1MDExODQxMjY1MTM5MQ==)

[Episode 160: ART History (Android Developers Backstage)](https://podcasts.google.com/feed/aHR0cHM6Ly9hZGJhY2tzdGFnZS5saWJzeW4uY29tL3Jzcw==/episode/NDE0N2JmY2ItYWNjNS00ZTU0LTk2YjUtODA5YjY0OGY4ZGJl)

[Episode 156: Android Runtime Classic (Dalvik) (Android Developers Backstage)](https://podcasts.google.com/feed/aHR0cHM6Ly9hZGJhY2tzdGFnZS5saWJzeW4uY29tL3Jzcw/episode/dGFnOmJsb2dnZXIuY29tLDE5OTk6YmxvZy0xMDUyMTA4NTQ3OTk4MDgyNDY1LnBvc3QtODA5NjE3MTEyNDY0MjU3NzE0NA?sa=X&ved=0CAUQkfYCahcKEwjYxsT9wNjwAhUAAAAAHQAAAAAQAg)

[Episode 83: The Deal of the ART (Android Developers Backstage)](https://podcasts.google.com/feed/aHR0cHM6Ly9hZGJhY2tzdGFnZS5saWJzeW4uY29tL3Jzcw/episode/dGFnOmJsb2dnZXIuY29tLDE5OTk6YmxvZy0xMDUyMTA4NTQ3OTk4MDgyNDY1LnBvc3QtNzQ1NDEzNDk3NjY2OTcyOTg1Mw?sa=X&ved=0CAUQkfYCahcKEwjYxsT9wNjwAhUAAAAAHQAAAAAQAg)

[proandroiddev.com/collecting-…](https://proandroiddev.com/collecting-the-garbage-a-brief-history-of-gc-over-android-versions-f7f5583e433c)


# 参考

https://juejin.cn/post/6966205309782065159#heading-0

