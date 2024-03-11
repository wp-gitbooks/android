内存管理?  （虚拟机如何使用内存？）

​        内存泄露、oom

​        gc

执行子系统?

程序编译与优化?

高效并发?（如何让java程序有更高的并发性？）

JVM的工作原理?  （结合实践）

java程序是如何运行的?



OSGI 模块化



# Java内存

![image-20210322122535902](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210322122536.png)

## 结构

JVM 内存结构是指：Java 虚拟机定义了若干种程序运行期间会使用的运行时数据区，其中有一些会随着虚拟机启动而创建，随着虚拟机退出而销毁，另一些则与线程一一对应，随着线程的开始而创建，随着线程的结束而销毁



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321184548.jpg)



线程：

​	线程共享

​	线程私有

异常：

StackOverflowError：java虚拟机栈、本地方法栈

OutOfMemoryError：java虚拟机栈、本地方法栈、方法区、堆



![image-20210321190505559](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321190505.png)



![image-20210321190542752](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321190727.png)

![image-20210321190659436](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321190659.png)





### JVM内存布局和相应控制

![image-20210321184909055](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321184909.png)

### 线程共享

#### Java堆(Heap)

Java 堆是所有线程共享的一块内存区域，它在虚拟机启动时 就会被创建，并且单个 JVM 进程有且仅有一个 Java 堆。Java 堆是用来存放对象实例及数组，也就是说我们代码中通过 new 关键字 new 出来的对象都存放在这里

![Java 堆内存结构](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321185249.png)

#### 方法区（Method Area）

它存储了每个类的结构信息，例如运行时常量池、字段、方法数据、构造函数和普通方法的字节码内容，还包括一些在类、实例、接口初始化时用到的特殊方法。



##### 常量池

String类的intern()方法





### 线程私有

跟线程同时创建，所以它跟线程有相同的生命周期

#### Java 虚拟机栈（JVM Stacks）

每一个方法在执行的同时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息，每一个方法从调用直至执行完成的过程，就对应着一个栈帧在 Java 虚拟机栈中的入栈到出栈的过程



**局部变量**表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、**对象引用**（reference 类型，它不等同于对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的引用指针，也可能指向一个代表对象的句柄或者其他与此对象相关的位置）和 returnAddress 类型（指向了一条字节码指令的地址）。

其中 64 位长度的 long 和 double 类型的数据会占用 2 个局部变量空间（Slot），其余的数据类型只占用 1 个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小



`Java` 虚拟栈中可能出现两种异常：

- `StackOverflowError`：线程请求的栈深度大于虚拟机所允许的深度
- `OutOfMemoryError`：虚拟机栈扩展时无法申请到足够的内存



#### 本地方法栈（Native Method Stacks）

本地方法栈（Native Method Stacks）与 Java 虚拟机栈所发挥的作用是非常相似的，其区别不过是 Java 虚拟机栈为虚拟机执行 Java 方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的 Native 方法服务。



#### 程序计数器（Program Counter Register）

程序计数器也是线程私有的，它只需要一块较小的内存空间，每条线程都要有一个独立的程序计数器，你可以把它看作当前线程所执行的字节码的**行号指示**器

字节码解释器工作时就是通过改变这个计数器的值来选取**下一条需要执行的字节码指令**，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成



- 如果线程正在执行 `Java` 方法，则计数器记录的是正在执行的虚拟机字节码指令的地址
- 如果执行 `native` 方法，则计数器为空



**程序计数器是唯一一个在Java虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域**。



### 特殊（直接内存）

直接内存(Direct Memory)不是虚拟机运行时数据区的一部分，也不是java虚拟机规范中的内存区域，但也可能导致OutOfMemoryError



NIO：DirectByteBuffer直接操作，避免Java堆和Native堆来回赋值数据



##对象访问

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210322123207.png)



### jvm层面引用(reference)

（1）强引用（Strong Reference）

（2）软引用（Soft Reference）

（3）弱引用（Weak Reference）

（4）、虚引用（Phantom Reference）



### 对象访问方式

主流的对象访问方式有两种：

#### 使用句柄

Java堆划分一块内存作为句柄池，reference中存储就是对象的句柄地址；

​    对象句柄包含两个地址：

> ​    （1）、在堆中分配的对象实例数据的地址；
>
> ​    （2）、这个对象类型数据地址；  

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321193827.png)



​    优点：对象移动时（垃圾回收时常见的动作），reference不需要修改，只改变句柄中实例数据指针；        



#### 使用直接指针

reference中存储就是在堆中分配的对象实例数据的地址；

​    而对象实例数据中需要有这个对象类型数据的相关信息（前面文章讨论了HotSpot使用对象头来存储对象类型数据地址）

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321193914.png)



## 内存分配策略(垃圾收集)

GC三件事：

​	1、哪些内存需要回收？

​    2、什么时候回收？

​    3、如何回收？

### 概述

#### 什么是垃圾回收

垃圾回收（Garbage Collection，GC），顾名思义就是释放垃圾占用的空间，防止内存泄露。有效的使用可以使用的内存，对内存堆中已经死亡的或者长时间没有使用的对象进行清除和回收。



#### 怎么定义垃圾

既然我们要做垃圾回收，首先我们得搞清楚垃圾的定义是什么，哪些内存是需要回收的。



##### 引用计数算法

引用计数算法（Reachability Counting）是通过在对象头中分配一个空间来保存该对象被引用的次数（Reference Count）。如果该对象被其它对象引用，则它的引用计数加 1，如果删除对该对象的引用，那么它的引用计数就减 1，当该对象的引用计数为 0 时，那么该对象就会被回收。

```
String m = new String("jack");
```

先创建一个字符串，这时候"jack"有一个引用，就是 m。

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321223155.png)

然后将 m 设置为 null，这时候"jack"的引用次数就等于 0 了，在引用计数算法中，意味着这块内容就需要被回收了。

```
m = null;
```

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321223253.png)



###### 问题：相互引用

```
public class ReferenceCountingGC {

    public Object instance;

    public ReferenceCountingGC(String name){}
}

public static void testGC(){

    ReferenceCountingGC a = new ReferenceCountingGC("objA");
    ReferenceCountingGC b = new ReferenceCountingGC("objB");

    a.instance = b;
    b.instance = a;

    a = null;
    b = null;
}

```

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321223408.png)



##### 可达性分析算法

可达性分析算法（Reachability Analysis）的基本思路是，通过一些被称为引用链（GC Roots）的对象作为起点，从这些节点开始向下搜索，搜索走过的路径被称为（Reference Chain)，当一个对象到 GC Roots 没有任何引用链相连时（即从 GC Roots 节点到该节点不可达），则证明该对象是不可用的。

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321223454.png)



###### GC ROOT

![image-20191207143050101](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321230756)

GC Root 的对象包括以下 4 种：

- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中 JNI（即一般说的 Native 方法）引用的对象



* 虚拟机栈（栈帧中的本地变量表）中引用的对象

```
此时的 s，即为 GC Root，当 s 置空时，localParameter 对象也断掉了与 GC Root 的引用链，将被回收。

public class StackLocalParameter {
    public StackLocalParameter(String name){}
}

public static void testGC(){
    StackLocalParameter s = new StackLocalParameter("localParameter");
    s = null;
}
```



* **方法区中类静态属性引用的对象**

  ```
  s 为 GC Root，s 置为 null，经过 GC 后，s 所指向的 properties 对象由于无法与 GC Root 建立关系被回收。
  
  
  而 m 作为类的静态属性，也属于 GC Root，parameter 对象依然与 GC root 建立着连接，所以此时 parameter 对象并不会被回收。
  
  public class MethodAreaStaicProperties {
      public static MethodAreaStaicProperties m;
      public MethodAreaStaicProperties(String name){}
  }
  
  public static void testGC(){
      MethodAreaStaicProperties s = new MethodAreaStaicProperties("properties");
      s.m = new MethodAreaStaicProperties("parameter");
      s = null;
  }
  ```

* **方法区中常量引用的对象**

  ```
  m 即为方法区中的常量引用，也为 GC Root，s 置为 null 后，final 对象也不会因没有与 GC Root 建立联系而被回收。
  
  
  public class MethodAreaStaicProperties {
      public static final MethodAreaStaicProperties m = MethodAreaStaicProperties("final");
      public MethodAreaStaicProperties(String name){}
  }
  
  public static void testGC(){
      MethodAreaStaicProperties s = new MethodAreaStaicProperties("staticProperties");
      s = null;
  }
  
  ```

* **本地方法栈中引用的对象**

  ![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321224107.png)





##### 引用

引用的使用场景

https://toutiao.io/posts/6wiqyz/preview

https://juejin.cn/post/6844904057602064391#heading-5



### 垃圾回收过程

#### 堆



![image-20191204225410181](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321230628)

#### 方法区

永久代垃圾回收主要回收两部分：废弃常量和无用的类

**废弃常量**同回收java堆类似。（以String对象是"acb"为例，没有地方引用到"abc"，会被回收）。常量池中其他类(接口)、方法、字段的符号引用类似。



**无用的类**

1、不存在该类的任何实例

2、加载改类的ClassLoader被回收

3、改类对应的java.lang.Class对应没有任何地方被引用，无法在如何地方通过反射调用



### 垃圾回收算法(怎么回收垃圾)

https://juejin.cn/post/6844904057602064391#heading-24

#### 概述

![image-20191207142817822](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321232044)

#### 标记-清除算法

##### 算法描述

- 标记阶段：标记处所有需要回收的对象；

- 清除阶段：标记完成后，统一回收所有被标记的对象；

  ![image-20210321233522306](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321233522.png)

##### 优点

还没想到

##### 不足

- 效率不高：标记和清除两个过程效率都不高；
- 空间问题：产生大量不连续的内存碎片，进而无法容纳大对象提早触发另一次GC。

#### 复制算法

##### 算法描述

- 将可用内存分为容量大小相等的两块，每次只使用其中一块；
- 当一块用完，就将存活着的对象复制到另一块，然后将这块全部内存清理掉；

![image-20210321233630316](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321233630.png)



##### 优点

- 不会产生不连续的内存碎片；
- 提高效率：
  - 回收：每次都是对整个半区进行回收；
  - 分配：分配时也不用考虑内存碎片问题，只要移动堆顶指针，按顺序分配内存即可。

##### 缺点

- 可用内存缩小为原来的一半了，适合GC过后只有少量存活的`新生代`，可以根据实际情况，将内存块大小比例适当调整；
- 如果存活对象数量比较大，复制性能会变得很差。

##### JVM中新生代的垃圾回收

如下图，分为新生代和老年代。其中新生代又分为一个Eden区和两个Survivor去(from区和to区)，默认Eden : from : to 比例为`8:1:1`。

可通过JVM参数：`-XX:SurvivorRatio`配置比例，`-XX:SurvivorRatio=8` 表示 `Eden区大小 / 1块Survivor区大小 = 8`。

**第一次Young GC**

![image-20210321233712650](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321233712.png)

当Eden区满的时候，触发第一次Young GC，把存活对象拷贝到Survivor的from区，清空Eden区。

**第二次Young GC**

![image-20210321233731385](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321233731.png)

再次触发Young GC，扫描Eden区和from区，把存活的对象复制到To区，清空Eden区和from区。如果此时Survivor区的空间不够了，就会提前把对象放入老年代。

默认的，这样来回交换15次后，如果对象最终还是存活，就放入老年代。

> 交换次数可以通过JVM参数`MaxTenuringThreshold`进行设置。

##### JVM内存模型

**JDK8 之前**

![image-20210321233755902](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321233755.png)

**JDK8**

![image-20210321233809645](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321233809.png)

如上图，JDK8的方法区实现变成了元空间，元空间在本地内存中。

**JVM内存相关参数：**

[JVM Parameters](https://www.javadevjournal.com/java/jvm-parameters/)

内存分配如何保证并发？

#### 标记-整理算法

##### 算法描述

- 标记过程与标记-清楚算法一样；
- 标记完成后，将存活对象向一端移动，然后直接清理掉边界以外的内存。

![image-20210321233855297](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321233855.png)

##### 优点

- 不会产生内存碎片；
- 不需要浪费额外的空间进行分配担保；

##### 不足

- 整理阶段存在效率问题，适合老年代这种垃圾回收频率不是很高的场景；

#### 分代收集算法

当前商业虚拟机都采用该算法。

- `新生代`：复制算法(CG后只有少量的对象存活)
- `老年代`：标记-整理算法 或者 标记-清理算法(GC后对象存活率高)

### 垃圾回收器

https://juejin.cn/post/6844904057602064391#heading-24



### 内存分配与回收策略

自动内存管理归结到两个问题：对象内存分配以及回收分配到对象的内存



#### 结构

![image-20210322103752009](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210322103752.png)



![image-20210322122818036](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210322122818.png)



![image-20210322122913072](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210322122913.png)



#### Eden 区

有将近 **98%的对象是朝生夕死**，所以针对这一现状，大多数情况下，对象会在新生代 Eden 区中进行分配，当 Eden 区没有足够空间进行分配时，虚拟机会发起一次 Minor GC，Minor GC 相比 Major GC 更频繁，回收速度也更快

通过 **Minor GC** 之后，Eden 会被清空，Eden 区中绝大部分对象会被回收，而那些无需回收的存活对象，将会进到 Survivor 的 From 区（若 From 区不够，则直接进入 Old 区）。



#### Survivor 区

Survivor 区相当于是 Eden 区和 Old 区的一个缓冲，类似于我们交通灯中的黄灯。Survivor 又分为 2 个区，一个是 From 区，一个是 To 区。每次执行 Minor GC，会将 Eden 区和 From 存活的对象放到 Survivor 的 To 区（如果 To 区不够，则直接进入 Old 区）。

##### 为啥需要？

不就是新生代到老年代么，直接 Eden 到 Old 不好了吗，为啥要这么复杂。想想如果没有 Survivor 区，Eden 区每进行一次 Minor GC，存活的对象就会被送到老年代，老年代很快就会被填满。而有很多对象虽然一次 Minor GC 没有消灭，但其实也并不会蹦跶多久，或许第二次，第三次就需要被清除。这时候移入老年区，很明显不是一个明智的决定。



所以，Survivor 的存在意义就是减少被送到老年代的对象，进而减少 Major GC 的发生。Survivor 的预筛选保证，只有经历 16 次 Minor GC 还能在新生代中存活的对象，才会被送到老年代

##### 为啥需要俩？

设置两个 Survivor 区最大的好处就是解决内存碎片化。



我们先假设一下，Survivor 如果只有一个区域会怎样。Minor GC 执行后，Eden 区被清空了，存活的对象放到了 Survivor 区，而之前 Survivor 区中的对象，可能也有一些是需要被清除的。问题来了，这时候我们怎么清除它们？在这种场景下，我们只能标记清除，而我们知道标记清除最大的问题就是内存碎片，在新生代这种经常会消亡的区域，采用标记清除必然会让内存产生严重的碎片化。因为 Survivor 有 2 个区域，所以每次 Minor GC，会将之前 Eden 区和 From 区中的存活对象复制到 To 区域。第二次 Minor GC 时，From 与 To 职责兑换，这时候会将 Eden 区和 To 区中的存活对象再复制到 From 区域，以此反复。



这种机制最大的好处就是，整个过程中，永远有一个 Survivor space 是空的，另一个非空的 Survivor space 是无碎片的。那么，Survivor 为什么不分更多块呢？比方说分成三个、四个、五个?显然，如果 Survivor 区再细分下去，每一块的空间就会比较小，容易导致 Survivor 区满，两块 Survivor 区可能是经过权衡之后的最佳方案



#### Old 区

老年代占据着 2/3 的堆内存空间，只有在 Major GC 的时候才会进行清理，每次 GC 都会触发“Stop-The-World”。内存越大，STW 的时间也越长，所以内存也不仅仅是越大就越好。由于复制算法在对象存活率较高的老年代会进行很多次的复制操作，效率很低，所以老年代这里采用的是标记 — 整理算法。



除了上述所说，在内存担保机制下，无法安置的对象会直接进到老年代，以下几种情况也会进入老年代。



#### 大对象

大对象指需要大量连续内存空间的对象，这部分对象不管是不是“朝生夕死”，都会直接进到老年代。这样做主要是为了避免在 Eden 区及 2 个 Survivor 区之间发生大量的内存复制。当你的系统有非常多“朝生夕死”的大对象时，得注意了



#### 长期存活对象

虚拟机给每个对象定义了一个对象年龄（Age）计数器。正常情况下对象会不断的在 Survivor 的 From 区与 To 区之间移动，对象在 Survivor 区中每经历一次 Minor GC，年龄就增加 1 岁。当年龄增加到 15 岁时，这时候就会被转移到老年代。当然，这里的 15，JVM 也支持进行特殊设置。



#### 动态对象年龄

虚拟机并不重视要求对象年龄必须到 15 岁，才会放入老年区，如果 Survivor 空间中相同年龄所有对象大小的总合大于 Survivor 空间的一半，年龄大于等于该年龄的对象就可以直接进去老年区，无需等你“成年”。



这其实有点类似于负载均衡，轮询是负载均衡的一种，保证每台机器都分得同样的请求。看似很均衡，但每台机的硬件不通，健康状况不同，我们还可以基于每台机接受的请求数，或每台机的响应时间等，来调整我们的负载均衡算法

#### 空间分配担保

##### 谁进行空间担保？

　　JVM使用分代收集算法，将堆内存划分为年轻代和老年代，两块内存分别采用不同的垃圾回收算法，空间担保指的是老年代进行空间分配担保

##### 什么是空间分配担保？

发生Minor GC前，JVM先检查老年代最大可用连续空间是否大于新生代所有对象的总空间

- 大于：空间足够，直接Minor GC；
- 小于：进行一次Full GC。

##### 为什么要进行空间担保？

是因为新生代采用**复制收集算法**，假如大量对象在Minor GC后仍然存活（最极端情况为内存回收后新生代中所有对象均存活），而Survivor空间是比较小的，这时就需要老年代进行分配担保，把Survivor无法容纳的对象放到老年代。**老年代要进行空间分配担保，前提是老年代得有足够空间来容纳这些对象**，但一共有多少对象在内存回收后存活下来是不可预知的，**因此只好取之前每次垃圾回收后晋升到老年代的对象大小的平均值作为参考**。使用这个平均值与老年代剩余空间进行比较，来决定是否进行Full GC来让老年代腾出更多空间

##### 

##### Minor Gc 和 Full GC 有什么不同呢？

针对 HotSpot VM 的实现，它里面的 GC 其实准确分类只有两大种：

部分收集 (Partial GC)：

- 新生代收集（Minor GC / Young GC）：只对新生代进行垃圾收集；
- 老年代收集（Major GC / Old GC）：只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集；
- 混合收集（Mixed GC）：对整个新生代和部分老年代进行垃圾收集。

整堆收集 (Full GC)：收集整个 Java 堆和方法区。



# 第二部分自动内存管理机制



## 第3章 垃圾收集器与内存分配策略

https://yq.aliyun.com/articles/753815

https://www.cnblogs.com/shoshana-kong/p/10572563.html

https://help.eclipse.org/2020-12/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Fconcepts%2Fgcroots.html&cp=37_2_3

https://blog.csdn.net/u010798968/article/details/72835255



android中实践：

https://jsonchao.github.io/2019/01/06/Android%E4%B8%BB%E6%B5%81%E4%B8%89%E6%96%B9%E5%BA%93%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E5%85%AD%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Leakcanary%E6%BA%90%E7%A0%81%EF%BC%89/

https://juejin.cn/post/6844904168075821064

https://juejin.cn/post/6844904090179207176#heading-13

https://developer.android.com/studio/profile/memory-profiler?hl=zh-cn



垃圾收集器的行为、优势、劣势？



GC做的事情：

1、哪些内存需要回收？

2、什么时候回收？

3、如何回收？



监控和调节





根搜索算法：GC Roots



新生代垃圾收集算法：

老年代垃圾收集算法：



垃圾收集器：

暂停所有线程；暂停时间

单线程、多线程





自动内存管理：给对象分配内存、回收分配给对象的内存



什么时候触发回收？

1.JVM空闲的时候 2.显式调用system.gc（） 3.空间满时

##### Scavenge GC

一般情况下，当新对象生成，并且在Eden申请空间失败时，就会触发Scavenge GC，对Eden区域进行GC， 清除非存活对象，并且把尚且存活的对象移动到Survivor区。然后整理Survivor的两个区。这种方式的GC是对 年轻代的Eden区进行，不会影响到年老代。因为大部分对象都是从Eden区开始的，同时Eden区不会分配的很 大，所以Eden区的GC会频繁进行。因而，一般在这里需要使用速度快、效率高的算法，使Eden去能尽快空闲 出来。

##### Full GC

对整个堆进行整理，包括Young、Tenured和Perm。Full GC因为需要对整个对进行回收，所以比Scavenge GC要慢，因此应该尽可能减少Full GC的次数。在对JVM调优的过程中，很大一部分工作就是对于FullGC的调 节。有如下原因可能导致Full GC：

- 年老代（Tenured）被写满
- 持久代（Perm）被写满
- System.gc()被显示调用



# 第三部分 虚拟机执行子系统

## 第6章 类文件结构

https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.7.21

https://blog.csdn.net/u010963948/article/details/90080056

https://wiki.jikexueyuan.com/project/java-vm/class.html

https://www.cnblogs.com/ysocean/p/11427535.html

https://juejin.cn/post/6844903512304795661#heading-0

https://blog.csdn.net/u010963948/article/details/90080056

https://zhuanlan.zhihu.com/p/25823310



https://wiki.jikexueyuan.com/project/java-vm/class-loading-mechanism.html





## 第8章 虚拟机字节码执行引擎





## 第9章 类加载及执行子系统的案例与实战

InvocationHandler 接口



动态代理：

https://blog.csdn.net/u013803262/article/details/53187363

https://segmentfault.com/a/1190000022789831