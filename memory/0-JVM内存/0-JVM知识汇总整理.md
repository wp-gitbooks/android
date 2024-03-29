# 线索
内存管理：内存创建、垃圾回收

类加载过程、类加载器



![image.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303192252766.png)

![image.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303192252926.png)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071458361.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111901024.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210803213908.jpg)


# 一、Java内存布局

## 1、Java内部布局全貌

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111902344.png)
JVM包含两个子系统和两个组件：

- 两个子系统为Class loader(类装载)、Execution engine(执行引擎)；
- 两个组件为Runtime data area(运行时数据区)、Native Interface(本地接口)。

各组件的功能大致如下：

- Class loader(类装载)：根据给定的全限定名类名(如：java.lang.Object)来装载class文件到Runtime data area中的method area。
- Execution engine（执行引擎）：执行classes中的指令。
- Native Interface(本地接口)：与native libraries交互，是其它编程语言交互的接口。
- Runtime data area(运行时数据区域)：这就是我们常说的JVM的内存。

## 2、Java内存模型工作机制

- 首先利用IDE集成开发工具编写Java源代码，源文件的后缀为.java；
- 再利用编译器(javac命令)将源代码编译成字节码文件，字节码文件的后缀名为.class；
- 运行字节码的工作是由解释器(java命令)来完成的。
- 接下来类加载器又将这些.class文件加载到JVM中
- 将类的.class文件中的二进制数据读入到内存中，将其放在运行时数据区的方法区内，然后在堆区创建一个 java.lang.Class对象，用来封装类在方法区内的数据结构。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111902299.png)
## 3、JVM 运行时数据区  [[JVM内存与结构]]
![image-20210321190505559](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210321190505.png)

Java 虚拟机在执行 Java 程序的过程中会把它管理的内存划分成若干个不同的数据区域。JDK. 1.8 和之前的版本略有不同，下面会介绍到。


**JDK 1.8 之前：**
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111903521.png)


![image.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303192251893.png)
**JDK 1.8 ：**
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111904667.png)

不同虚拟机的运行时数据区可能略微有所不同，但都会遵从 Java 虚拟机规范， Java 虚拟机规范规定的区域分为以下 5 个部分，**其中线程私有程序计数器、虚拟机栈、本地方法栈；线程共享堆、方法区、直接内存。**

### （1）程序计数器（Program Counter Register）

程序计数器是线程私有的属性，其主要有两个作用：

- 字节码解释器通过改变程序计数器来依次读取指令，从而实现代码的流程控制，如：顺序执行、选择、循环、异常处理。
- 在多线程的情况下，程序计数器用于记录当前线程执行的位置，从而当线程被切换回来的时候能够知道该线程上次运行到哪儿了

**注意：程序计数器是唯一一个不会出现 OutOfMemoryError 的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡。**

### （2）Java 虚拟机栈（Java Virtual Machine Stacks）

与程序计数器一样，Java 虚拟机栈也是线程私有的，Java 内存可以粗糙的区分为堆内存（Heap）和栈内存 (Stack),**其中栈就是现在说的虚拟机栈**，或者说是虚拟机栈中局部变量表部分。Java虚拟机栈中存放局部变量表、操作数栈、动态链接、方法出口信息。

**Java 虚拟机栈会出现两种错误：StackOverFlowError 和 OutOfMemoryError。**

- **StackOverFlowError：** 若 Java 虚拟机栈的内存大小不允许动态扩展，那么当线程请求栈的深度超过当前 Java 虚拟机栈的最大深度的时候，就抛出 StackOverFlowError 错误。
- **OutOfMemoryError：** 若 Java 虚拟机栈的内存大小允许动态扩展，且当线程请求栈时内存用完了，无法再动态扩展了，此时抛出 OutOfMemoryError 错误。

### （3）本地方法栈（Native Method Stack）

和虚拟机栈所发挥的作用非常相似，区别是：**虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。** 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

方法执行完毕后相应的栈帧也会出栈并释放内存空间，也会出现 **StackOverFlowError 和OutOfMemoryError** 两种错误。

### （4）Java 堆（Java Heap）

Java 虚拟机所管理的内存中最大的一块，线程共享，**此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。**Java 堆是垃圾收集器管理的主要区域，因此也被称作**GC 堆（Garbage Collected Heap）**。

堆这里最容易出现的就是 OutOfMemoryError 错误，并且出现这种错误之后的表现形式还会有几种，比如：

1. **`OutOfMemoryError: GC Overhead Limit Exceeded`** ：当JVM花太多时间执行垃圾回收并且只能回收很少的堆空间时，就会发生此错误。
2. **`java.lang.OutOfMemoryError: Java heap space`** :假如在创建新的对象时, 堆内存中的空间不足以存放新创建的对象, 就会引发`java.lang.OutOfMemoryError: Java heap space` 错误。(和本机物理内存无关，和你配置的内存大小有关！)

### （5）方法区（Methed Area）

线程共享的一块区域，它用于存储已被虚拟机加载的类信息、常量、**静态变量**、即时编译器编译后的代码等数据。

**JDK 8 版本之后方法区（HotSpot 的永久代）被彻底移除了（JDK1.7 就已经开始了），取而代之是元空间，元空间使用的是直接内存。**

### （6）运行时常量池（Runtime Constant Pool）

用于存放编译期生成的各种字面量和符号引用，当常量池无法再申请到内存时会抛出 OutOfMemoryError 错误。

- **JDK1.7之前运行时常量池逻辑包含字符串常量池存放在方法区, 此时hotspot虚拟机对方法区的实现为永久代**
- **JDK1.7 字符串常量池被从方法区拿到了堆中, 这里没有提到运行时常量池,也就是说字符串常量池被单独拿到堆,运行时常量池剩下的东西还在方法区, 也就是hotspot中的永久代** 。
- **JDK1.8 hotspot移除了永久代用元空间(Metaspace)取而代之, 这时候字符串常量池还在堆, 运行时常量池还在方法区, 只不过方法区的实现从永久代变成了元空间(Metaspace)**

### （7）直接内存（Direct Memory）

直接内存并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用。而且也可能导致 OutOfMemoryError 错误出现。

> JDK1.4 中新加入的 **NIO(New Input/Output) 类**，引入了一种基于**通道（Channel）** 与**缓存区（Buffer）** 的 I/O 方式，它可以直接使用 Native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样就能在一些场景中显著提高性能，因为**避免了在 Java 堆和 Native 堆之间来回复制数据**。

本机直接内存的分配不会受到 Java 堆的限制，但是，既然是内存就会受到本机总内存大小以及处理器寻址空间的限制。

### （8）补充：堆和栈的区别   [[Java堆和栈]]

- 物理地址：

- - 堆的物理地址分配对对象是不连续的。因此性能慢些。
  - 栈使用的是数据结构中的栈，先进后出的原则，物理地址分配是连续的。所以性能快。

- 内存分配：

- - 堆因为是不连续的，所以分配的内存是在`运行期`确认的，因此大小不固定。一般堆大小远远大于栈。
  - 栈是连续的，所以分配的内存大小要在`编译期`就确认，大小是固定的。

- 存放的内容：

- - 堆存放的是**对象的实例和数组**。因此该区更关注的是数据的存储
  - 栈存放：**局部变量，操作数栈**，返回结果。该区更关注的是程序方法的执行。

- 静态变量放在方法区，静态的对象还是放在堆。

- **程序的可见度**：

- - 堆对于整个应用程序都是**共享、可见**的。
  - 栈只对于线程是可见的。所以也是**线程私有**。他的生命周期和线程相同。

## 4、JVM中对象的创建过程

下图便是 Java 对象的创建过程，图后会详细说明每一步的作用：
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111905026.png)
### （1）类加载检查

虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能在常量池中定位到这个类的符号引用，并且检查这个符号引用代表的类是否已被加载过、解析和初始化过。如果没有，那必须先执行相应的类加载过程。

### （2）分配内存   [[变量内存分配和回收]]

在类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需的内存大小在类加载完成后便可确定，为对象分配空间的任务等同于把一块确定大小的内存从 Java 堆中划分出来。

#### 1）内存分配方式

**分配方式有 “指针碰撞” 和 “空闲列表” 两种，选择那种分配方式由 Java 堆是否规整决定，而 Java 堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定。**

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111905618.png)

#### 2）内存分配并发问题

对象的创建在虚拟机中是一个非常频繁的行为，哪怕只是修改一个指针所指向的位置，在并发情况下也是不安全的，可能出现正在给对象 A 分配内存，指针还没来得及修改，对象 B 又同时使用了原来的指针来分配内存的情况。解决这个问题有两种方案：

- 对分配内存空间的动作进行同步处理（采用 CAS + 失败重试来保障更新操作的原子性）；
- 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在 Java 堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer, TLAB）。哪个线程要分配内存，就在哪个线程的 TLAB 上分配。只有 TLAB 用完并分配新的 TLAB 时，才需要同步锁。通过-XX:+/-UserTLAB参数来设定虚拟机是否使用TLAB。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111905664.png)

### （3）初始化零值

内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），这一步操作保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

### （4）设置对象头

初始化零值完成之后，虚拟机要对对象进行必要的设置，**例如这个对象是那个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息。** 这些信息存放在对象头中。另外，根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。

### （5）执行初始化init方法

在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，但从 Java 程序的视角来看，对象创建才刚开始，在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，但从 Java 程序的视角来看，对象创建才刚开始， 方法还没有执行，所有的字段都还为零。**所以一般来说，执行 new 指令之后会接着执行方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来。** 方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来。

## 5、对象的内存布局

在 Hotspot 虚拟机中，对象在内存中的布局可以分为 3 块区域：**对象头**、**实例数据**和**对齐填充**。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111906699.png)

- **对象头：**对象头分为`Mark Word`和`Class Metadata Addresss`两个部分， `Mark Word`存储对象的hashCode、锁信息或者分代年龄GC等标志等信息。`Class Metadata Addresss`存放指向类元数据的指针，JVM通过这个指针确定该对象是那个类的实列。
- **实例数据：**存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐
- **对齐填充：**由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这点了解即可。

**Java对象内存布局中最重要的一块应该就是对象头中的`Mark Word`部分了，他涉及到了hash值、锁状态、分代年龄等许多非常重要的内容，下面就来详细捋一下：**

这部分主要用来存储对象自身的运行时数据，如hashCode、GC分代年龄等。`mark word`的位长度为JVM的一个Word大小，也就是说32位JVM的`Mark word`为32位，64位JVM为64位。为了让一个字大小存储更多的信息，JVM将字的最低两个位设置为标记位，不同标记位下的Mark Word示意如下：

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111906354.png)

```
|-------------------------------------------------------|--------------------|
|                  Mark Word (32 bits)                  |       State        |
|-------------------------------------------------------|--------------------|
| identity_hashcode:25 | age:4 | biased_lock:1 | lock:2 |       Normal       |
|-------------------------------------------------------|--------------------|
|  thread:23 | epoch:2 | age:4 | biased_lock:1 | lock:2 |       Biased       |
|-------------------------------------------------------|--------------------|
|               ptr_to_lock_record:30          | lock:2 | Lightweight Locked |
|-------------------------------------------------------|--------------------|
|               ptr_to_heavyweight_monitor:30  | lock:2 | Heavyweight Locked |
|-------------------------------------------------------|--------------------|
|                                              | lock:2 |    Marked for GC   |
|-------------------------------------------------------|--------------------|
```

上面分别给出了32位和64位JVM中markword的区别，大致可以发现除了位数，基本上都是一样了。

这里以32位JVM为例，64位的情况下是一样的，32位JVM一个字是32为，64位JVM一个字是64为，JVM均用一个字的大小记录当前Mark Word中的信息。

### （1）Normal无所状态

- `identity_hashcode`: 25位的对象标识Hash码，采用延迟加载技术。调用方法`System.identityHashCode()`计算，并会将结果写到该对象头中。**当对象被锁定时，该值会移动到管程Monitor中。**也就是说当对象加锁之后，前25位将不再是对象的hashCode。

- `age:`4位的Java对象年龄。在GC中，如果对象在Survivor区复制一次，年龄增加1。当对象达到设定的阈值时，将会晋升到老年代。默认情况下，并行GC的年龄阈值为15，并发GC的年龄阈值为6。**由于age只有4位，所以最大值为15**，这就是`-XX:MaxTenuringThreshold`选项最大值为15的原因。

- `biased_lock:`对象是否启用偏向锁标记，只占1个二进制位。为1时表示对象启用偏向锁，为0时表示对象没有偏向锁。

- `lock:`2位的锁状态标记位，由于希望用尽可能少的二进制位表示尽可能多的信息，所以设置了lock标记。该标记的值不同，整个mark word表示的含义不同。

  | biased_lock | lock· | 状态     |
  | :---------- | :---- | :------- |
  | 0           | 01    | 无锁     |
  | 1           | 01    | 偏向锁   |
  | 0           | 00    | 轻量级锁 |
  | 0           | 10    | 重量级锁 |
  | 0           | 11    | GC标记   |

### （2）Biased偏向锁状态

- `thread:` 23位表示持有偏向锁的线程ID。
- `epoch:` 2位，偏向时间戳,达到一定数量之后就会升级为轻量级锁。
- `age`、`biased_loc`、`lock`跟无锁的状态一致。

### （3）Lightweight Locked轻量级锁状态

- `ptr_to_lock_record:`30位指向栈中锁记录的指针。
- 剩下的两位用于标志当前的锁状态

### （4）Heavyweight Locked重量级锁状态

- `ptr_to_heavyweight_monitor:`30位指向管程Monitor的指针。
- 剩下的两位用于标志当前的锁状态

## 6、对象访问定位  [[对象访问]]
[[Java堆中对象分配、布局和访问的全过程]]

`Java`程序需要通过 `JVM` 栈上的引用访问堆中的具体对象。对象的访问方式取决于 `JVM` 虚拟机的实现。目前主流的访问方式有 **句柄** 和 **直接指针** 两种方式。

> **指针：** 指向对象，代表一个对象在内存中的起始地址。
>
> **句柄：** 可以理解为指向指针的指针，维护着对象的指针。句柄不直接指向对象，而是指向对象的指针（句柄不发生变化，指向固定内存地址），再由对象的指针指向对象的真实内存地址。

### （1）句柄访问

`Java`堆中划分出一块内存来作为**句柄池**，引用中存储对象的**句柄地址**，而句柄中包含了**对象实例数据**与**对象类型数据**各自的**具体地址**信息，具体构造如下图所示：

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111906839.png)


**优势**：引用中存储的是**稳定**的句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变**句柄中**的**实例数据指针**，而**引用**本身不需要修改。

### （2）直接指针

如果使用**直接指针**访问，**引用** 中存储的直接就是**对象地址**，那么`Java`堆对象内部的布局中就必须考虑如何放置访问**类型数据**的相关信息。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111907886.png)

**优势**：速度更**快**，节省了**一次指针定位**的时间开销。由于对象的访问在`Java`中非常频繁，因此这类开销积少成多后也是非常可观的执行成本。HotSpot 中采用的就是这种方式。

# 二、Java垃圾回收  [[0-垃圾回收-内存分配]]

垃圾回收主要就是防止JVM溢出而存在的一套JVM自动回收垃圾机制，主要需要弄明白下面几个问题：

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111907942.png)

## 1、内存如何分配和回收的

### （1）Java内存分配模型

Java 的自动内存管理主要是针对对象内存的回收和对象内存的分配。同时，Java 自动内存管理最核心的功能是 堆 内存中对象的分配与回收。

Java 堆是垃圾收集器管理的主要区域，因此也被称作**GC 堆（Garbage Collected Heap）**。从垃圾回收的角度，由于现在收集器基本都采用分代垃圾收集算法，所以 Java 堆还可以细分为：新生代和老年代：再细致一点有：Eden 空间、From Survivor、To Survivor 空间等。**进一步划分的目的是更好地回收内存，或者更快地分配内存。**

**堆空间的基本结构：**

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111907736.png)

### （2）Java堆内存分配策略

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111908714.png)

- 在堆中，如果待分配的对象所需内存大于eden区大小，那么将直接送入老年代。

- 如果eden区可以放下，会经历一下的分配过程：

- - 对象都会首先在 Eden 区域分配
  - 在一次新生代垃圾回收后，如果对象还存活，则会进入 s1("To")，并且对象的年龄还会加 1（初始为0）
  - 当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。
  - 经过这次GC后，Eden区和"From"区已经被清空。这个时候，"From"和"To"会交换他们的角色，也就是新的"To"就是上次GC前的“From”，新的"From"就是上次GC前的"To"（下文再详细阐述）

### （3）新生代内存中，为什么要有Survivor区域

- 如果没有Survivor，Eden区每进行一次Minor GC（发生在新生代的垃圾回收），存活的对象就会被送到老年代。老年代很快被填满，触发Major GC（因为Major GC一般伴随着Minor GC，也可以看做触发了Full GC）。老年代的内存空间远大于新生代，进行一次Full GC消耗的时间比Minor GC长得多。你也许会问，执行时间长有什么坏处？频发的Full GC消耗的时间是非常可观的，这一点会影响大型程序的执行和响应速度，更不要说某些连接会因为超时发生连接错误了。
- 从上面可以看出来，Survivor区域带来的最大的优势就是**防止老年代被很快填满，从而增大老年代垃圾回收时间上的浪费**

好，那我们来想想在没有Survivor的情况下，有没有什么解决办法，可以避免上述情况：

- 增加老年代空间 ，更多存活对象才能填满老年代。降低Full GC频率 随着老年代空间加大，一旦发生Full GC，执行所需要的时间更长
- 减少老年代空间 Full GC所需时间减少 老年代很快被存活对象填满，Full GC频率增加。显而易见，没有Survivor的话，上述两种解决方案都不能从根本上解决问题。

### （4）为什么要设置两个Survivor区
**这个问题也就是复制算法的原理，堆中新生代采用的就是复制算法，下面来看一下它的魅力：**

设置两个Survivor区最大的好处就是**解决了碎片化**，我们来分析一下：

**1）首先使用单个Survivor区**

- 刚刚新建的对象在Eden中，一旦Eden满了，触发一次Minor GC，Eden中的存活对象就会被移动到Survivor区。这样继续循环下去，下一次Eden满了的时候，问题来了，此时进行Minor GC，Eden和Survivor各有一些存活对象，如果此时把Eden区的存活对象硬放到Survivor区，很明显这两部分对象所占有的内存是不连续的，也就导致了内存碎片化。
- 碎片化带来的风险是极大的，严重影响JAVA程序的性能。堆空间被散布的对象占据不连续的内存，最直接的结果就是，堆中没有足够大的连续内存空间，接下去如果程序需要给一个内存需求很大的对象分配内存，就会造成很可怕的结果。下面用图简单说明一下只有一个Survivor会出现什么样的情况

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111908053.png)

- 第一次GC发生之前，Survivor区为空， GC过后， Eden清空， Survivor填上

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111908183.png)

- 第二次GC的时候，Survivor区域中有部分被标记清除，Eden又加入了一些新的元素， 那么继续清空如下，可以看到此轮GC过后，Survivor区因为此轮也清楚了两个红点，所以产生了两个碎片，而Eden区新加入的四个绿点紧跟着之前的Survivor加入

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111909926.png)

- 第三次GC的时候，同样的Eden区域和Survivor区域都产生了要回收的对象，看看这会出现了什么样的情况

经过上面的GC，可以看到最终再Survivor区出现了大量的碎片，那么向解决这个问题的最好的方式就是使用两个Survivor区。

**2）使用两个Survivor区**

咱们再来看看，用两个Survivor会出现什么样的情况。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111909895.png)

- 首先第一次GC，将Eden区红色全部干掉，绿色全部扔到from里面区

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111909165.png)

- 这里简单看一下第二次GC是怎么进行操作的，首先Eden区和from区这个时候又出现了许多红点（带清理的对象），这次JVM的操作就不是直接再Survivor区域上将对象干掉，这次他会收先将from区域里面的红点全部干点，然后剩余的绿点顺位进入to区域，eden区域同样按这个原理清空， 放入to区域之后，再将from区域和to区域交换，始终保证to区域是空闲的状态，这样就可以非常完美的解决碎片化问题。

## 2、哪些垃圾需要回收

堆中几乎放着所有的对象实例，对堆垃圾回收前的第一步就是要判断那些对象已经死亡（即不能再被任何途径使用的对象）。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111910268.png)

### （1）怎么判断对象已死亡

判断对象是否已经死亡通常有两个方法，引用计数法和可达性分析算法

#### 1）引用计数法

给对象中添加一个引用计数器，每当有一个地方引用它，计数器就加 1；当引用失效，计数器就减 1；任何时候计数器为 0 的对象就是不可能再被使用的。

**这个方法实现简单，效率高，但是目前主流的虚拟机中并没有选择这个算法来管理内存，其最主要的原因是它很难解决对象之间相互循环引用的问题。**

#### 2）可达性分析算法

这个算法的基本思想就是通过一系列的称为 **“GC Roots”** 的对象作为起点，从这些节点开始向下搜索，节点所走过的路径称为引用链，当一个对象到 GC Roots 没有任何引用链相连的话，则证明此对象是不可用的。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111910565.png)

### （2）四种引用是怎么进行垃圾回收的

JDK1.2 以后，Java 对引用的概念进行了扩充，将引用分为强引用、软引用、弱引用、虚引用四种（引用强度逐渐减弱）

#### 1）强引用（StrongReference）

有强引用的对象，垃圾回收器绝不会回收它，当内存空间不足，Java 虚拟机宁愿抛出 OutOfMemoryError 错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题。

#### 2）软引用（SoftReference）

如果内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存。常用作于告诉缓存

#### 3）弱引用（WeakReference）

在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。

#### 4）虚引用（PhantomReference）

与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。虚引用的用途是在 gc 时返回一个通知。

特别注意，在程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多，这是因为**软引用可以加速 JVM 对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生**。

### （3）如何判断一个常量是废弃常量

假如在常量池中存在字符串 "abc"，如果当前没有任何 String 对象引用该字符串常量的话，就说明常量 "abc" 就是废弃常量，如果这时发生内存回收的话而且有必要的话，"abc" 就会被系统清理出常量池。

### （4）如何判断一个类是无用的类

类需要同时满足下面 3 个条件才能算是 **“无用的类”** ：

- 该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例。
- 加载该类的 ClassLoader 已经被回收。
- 该类对应的 java.lang.Class 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

虚拟机可以对满足上述 3 个条件的无用类进行回收，这里说的仅仅是“可以”，而并不是和对象一样不使用了就会必然被回收。

## 3、什么时候回收

### （1）分代垃圾回收器工作流程

分代回收器有两个分区：老生代和新生代，新生代默认的空间占比总空间的 1/3，老生代的默认占比是 2/3。

新生代使用的是复制算法，新生代里有 3 个分区：Eden、To Survivor、From Survivor，它们的默认占比是 8:1:1，它的执行流程如下：

- 把 Eden + From Survivor 存活的对象放入 To Survivor 区；
- 清空 Eden 和 From Survivor 分区；
- From Survivor 和 To Survivor 分区交换，From Survivor 变 To Survivor，To Survivor 变 From Survivor。

每次在 From Survivor 到 To Survivor 移动时都存活的对象，年龄就 +1，当年龄到达 15（默认配置是 15）时，升级为老生代。大对象也会直接进入老生代。

老生代当空间占用到达某个值之后就会触发全局垃圾收回，一般使用标记整理的执行算法。以上这些循环往复就构成了整个分代垃圾回收的整体执行流程。

### （2）Minor GC和Major GC内存回收策略

多数情况，对象都在新生代 Eden 区分配。当 Eden 区分配没有足够的空间进行分配时，虚拟机将会发起一次 Minor GC。如果本次 GC 后还是没有足够的空间，则将启用分配担保机制在老年代中分配内存。

这里我们提到 Minor GC，如果你仔细观察过 GC 日常，通常我们还能从日志中发现 Major GC/Full GC。

- **Minor GC** 是指发生在新生代的 GC，因为 Java 对象大多都是朝生夕死，所有 Minor GC 非常频繁，一般回收速度也非常快；
- **Major GC/Full GC** 是指发生在老年代的 GC，出现了 Major GC 通常会伴随至少一次 Minor GC。Major GC 的速度通常会比 Minor GC 慢 10 倍以上。

### （3）不可达的对象并非“非死不可”

- 即使在可达性分析法中不可达的对象，也并非是“非死不可”的，这时候它们暂时处于“缓刑阶段”，要真正宣告一个对象死亡，至少要经历两次标记过程；
- 可达性分析法中不可达的对象被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行 finalize 方法。
- 当对象没有覆盖 finalize 方法，或 finalize 方法已经被虚拟机调用过时，虚拟机将这两种情况视为没有必要执行。
- 被判定为需要执行的对象将会被放在一个队列中进行第二次标记，除非这个对象与引用链上的任何一个对象建立关联，否则就会被真的回收。

### （4）可以主动通知虚拟机进行垃圾回收吗

可以。程序员可以手动执行System.gc()，通知GC运行，但是Java语言规范并不保证GC一定会执行。

## 4、如何回收

### （1）垃圾收集算法

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111910223.png)

#### 1）标记清除算法

该算法分为“标记”和“清除”阶段：首先比较出所有需要回收的对象，在标记完成后统一回收掉所有被标记的对象。它是最基础的收集算法，后续的算法都是对其不足进行改进得到。这种垃圾收集算法会带来两个明显的问题：

- **效率问题**
- **空间问题（标记清除后会产生大量不连续的碎片）**

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111910348.png)

#### 2）复制算法

为了解决效率问题，“复制”收集算法出现了。它可以将内存分为大小相同的两块，每次使用其中的一块。当这一块的内存使用完后，就将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉。这样就使每次的内存回收都是对内存区间的一半进行回收。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111911425.png)


#### 3）标记整理算法

根据老年代的特点提出的一种标记算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一端移动，然后直接清理掉端边界以外的内存。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111911014.png)

#### 4）分代收集算法

虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。一般将 java 堆分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。

- 在新生代使用复制算法
- 在老年代使用标记整理算法

### （2）垃圾收集器

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111911230.png)

**如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。**

下图展示了7种作用于不同分代的收集器，其中用于回收新生代的收集器包括Serial、PraNew、Parallel Scavenge，回收老年代的收集器包括Serial Old、Parallel Old、CMS，还有用于回收整个Java堆的G1收集器。不同收集器之间的连线表示它们可以搭配使用。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111912426.png)


- Serial收集器（复制算法): 新生代单线程收集器，标记和清理都是单线程，优点是简单高效；

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111912775.png)

- ParNew收集器 (复制算法): 新生代收并行集器，实际上是Serial收集器的多线程版本，在多核CPU环境下有着比Serial更好的表现；

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111912861.png)

- Parallel Scavenge收集器 (复制算法): 新生代并行收集器，追求高吞吐量，高效利用 CPU。吞吐量 = 用户线程时间/(用户线程时间+GC线程时间)，高吞吐量可以高效率的利用CPU时间，尽快完成程序的运算任务，适合后台应用等对交互相应要求不高的场景；

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111912779.png)

- Serial Old收集器 (标记-整理算法): 老年代单线程收集器，Serial收集器的老年代版本；
- Parallel Old收集器 (标记-整理算法)：老年代并行收集器，吞吐量优先，Parallel Scavenge收集器的老年代版本；
- CMS(Concurrent Mark Sweep)收集器（标记-清除算法）：老年代并行收集器，它非常符合在注重用户体验的应用上使用，以获取最短回收停顿时间为目标的收集器，具有高并发、低停顿的特点，追求最短GC回收停顿时间。
- G1(Garbage First)收集器 (标记-整理算法)：Java堆并行收集器，G1收集器是JDK1.7提供的一个新收集器，G1收集器基于“标记-整理”算法实现，也就是说不会产生内存碎片。此外，G1收集器不同于之前的收集器的一个重要特点是：G1回收的范围是整个Java堆(包括新生代，老年代)，而前六种收集器回收的范围仅限于新生代或老年代。

因为CMS和G1收集器相对比较特殊，下面单独介绍一下他们

### （3）CMS收集器

**CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。它非常符合在注重用户体验的应用上使用。**

**CMS（Concurrent Mark Sweep）收集器是 HotSpot 虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。**

从名字中的**Mark Sweep**这两个词可以看出，CMS 收集器是一种 **“标记-清除”算法**实现的，它的运作过程相比于前面几种垃圾收集器来说更加复杂一些。整个过程分为四个步骤：

- **初始标记：** 暂停所有的其他线程，并记录下直接与 root 相连的对象，速度很快 ；

- **并发标记：** 同时开启 GC 和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。因为用户线程可能会不断的更新引用域，所以 GC 线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。

- **重新标记：** 重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短

- **并发清除：** 开启用户线程，同时 GC 线程开始对未标记的区域做清扫。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111913819.png)

从它的名字就可以看出它是一款优秀的垃圾收集器，主要优点：**并发收集、低停顿**。但是它有下面三个明显的缺点：

- **对 CPU 资源敏感；**
- **无法处理浮动垃圾；**
- **它使用的回收算法-“标记-清除”算法会导致收集结束时会有大量空间碎片产生。**

### （4）G1收集器

**G1 (Garbage-First) 是一款面向服务器的垃圾收集器,主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足 GC 停顿时间要求的同时,还具备高吞吐量性能特征.**

被视为 JDK1.7 中 HotSpot 虚拟机的一个重要进化特征。它具备一下特点：

- **并行与并发**：G1 能充分利用 CPU、多核环境下的硬件优势，使用多个 CPU（CPU 或者 CPU 核心）来缩短 Stop-The-World 停顿时间。部分其他收集器原本需要停顿 Java 线程执行的 GC 动作，G1 收集器仍然可以通过并发的方式让 java 程序继续执行。
- **分代收集**：虽然 G1 可以不需要其他收集器配合就能独立管理整个 GC 堆，但是还是保留了分代的概念。
- **空间整合**：与 CMS 的“标记--清理”算法不同，G1 从整体来看是基于“标记整理”算法实现的收集器；从局部上来看是基于“复制”算法实现的。
- **可预测的停顿**：这是 G1 相对于 CMS 的另一个大优势，降低停顿时间是 G1 和 CMS 共同的关注点，但 G1 除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为 M 毫秒的时间片段内。

G1 收集器的运作大致分为以下几个步骤：

- **初始标记**
- **并发标记**
- **最终标记**
- **筛选回收**

**G1 收集器在后台维护了一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region(这也就是它的名字 Garbage-First 的由来)**。这种使用 Region 划分内存空间以及有优先级的区域回收方式，保证了 G1 收集器在有限时间内可以尽可能高的收集效率（把内存化整为零）。

------

# 三、类的生命周期  [[1-类加载过程]]

## 1、类完整的生命周期

一个类的完整生命周期如下：

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111913727.png)

Class 文件需要加载到虚拟机中之后才能运行和使用，那么虚拟机是如何加载这些 Class 文件呢？

系统加载 Class 类型的文件主要三步:**加载->连接->初始化**。连接过程又可分为三步:**验证->准备->解析**。

下面开始一步一步分析类加载的过程：

## 2、加载阶段

类的加载过程主要完成三件事：

- **通过全类名获取定义此类的二进制字节流（获取.class文件的字节流）**
- **将字节流上面所代表的静态存储结构转换为方法区的运行时数据结构**
- **在内存中生成一个代表该类的Class对象，作为方法区这些数据的访问入口**

值得注意的是，加载阶段和连接阶段的部分内容是交叉进行的，加载阶段尚未结束，连接阶段可能就已经开始了。

## 3、验证阶段

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111913573.png)


验证阶段主要就是对文件格式，元数据，字节码以及符号应用的一些验证，个人感觉跟编译检查一样的工作

## 4、准确阶段

**准备阶段是正式为类变量分配内存并设置类变量初始值的阶段**，这些内存都将在方法区中分配。对于该阶段有以下几点需要注意：

- 这时候进行内存分配的仅包括类变量（static），而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在 Java 堆中。
- 这里所设置的初始值"通常情况"下是数据类型默认的零值（如0、0L、null、false等），比如我们定义了`public static int value=111` ，那么 value 变量在准备阶段的初始值就是 0 而不是111（初始化阶段才会赋值）。特殊情况：比如给 value 变量加上了 fianl 关键字`public static final int value=111` ，那么准备阶段 value 的值就被赋值为 111。

## 5、解析阶段

**虚拟机将常量池中的符号引用替换成直接引用的过程。符号引用就理解为一个标示，而在直接引用直接指向内存中的地址；**

**符号引号和直接引用的区别：**

> 符号引用（Symbolic References）：
>
> - 符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能够无歧义的定位到目标即可。例如，在Class文件中它以CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等类型的常量出现。
> - 符号引用与虚拟机的内存布局无关，引用的目标并不一定加载到内存中。在Java中，一个java类将会编译成一个class文件。在编译时，java类并不知道所引用的类的实际地址，因此只能使用符号引用来代替。比如org.simple.People类引用了org.simple.Language类，在编译时People类并不知道Language类的实际内存地址，因此只能使用符号org.simple.Language（假设是这个，当然实际中是由类似于CONSTANT_Class_info的常量来表示的）来表示Language类的地址。
> - 各种虚拟机实现的内存布局可能有所不同，但是它们能接受的符号引用都是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。
>
> 直接引用（Direct References）：
>
> - 直接指向目标的指针（比如，指向“类型”【Class对象】、类变量、类方法的直接引用可能是指向方法区的指针）
> - 相对偏移量（比如，指向实例变量、实例方法的直接引用都是偏移量）
> - 一个能间接定位到目标的句柄
>
> 直接引用是和虚拟机的布局相关的，同一个符号引用在不同的虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经被加载入内存中了。

## 6、初始化阶段

**对静态变量和静态代码块执行初始化工作。**

初始化是类加载的最后一步，也是真正执行类中定义的 Java 程序代码(字节码)，初始化阶段是执行类构造器 方法的过程。对于构造方法的调用，虚拟机会自己确保其在多线程环境中的安全性。因为构造方法是带锁线程安全，所以在多线程环境下进行类初始化的话可能会引起死锁，并且这种死锁很难被发现。

对于初始化阶段，虚拟机严格规范了有且只有5种情况下，必须对类进行初始化(只有主动去使用类才会初始化类)：

（1）当遇到 new 、 getstatic、putstatic或invokestatic 这4条直接码指令时，比如 new 一个类，读取一个静态字段(未被 final 修饰)、或调用一个类的静态方法时。

- 当jvm执行new指令时会初始化类。即当程序创建一个类的实例对象。
- 当jvm执行getstatic指令时会初始化类。即程序访问类的静态变量(不是静态常量，常量会被加载到运行时常量池)。
- 当jvm执行putstatic指令时会初始化类。即程序给类的静态变量赋值。
- 当jvm执行invokestatic指令时会初始化类。即程序调用类的静态方法。

（2）使用 `java.lang.reflect` 包的方法对类进行反射调用时如Class.forname("…"),newInstance()等等。，如果类没初始化，需要触发其初始化。

（3）初始化一个类，如果其父类还未初始化，则先触发该父类的初始化。

（4）当虚拟机启动时，用户需要定义一个要执行的主类 (包含 main 方法的那个类)，虚拟机会先初始化这个类。

（5）MethodHandle和VarHandle可以看作是轻量级的反射调用机制，而要想使用这2个调用， 就必须先使用findStaticVarHandle来初始化要调用的类。

## 7、卸载过程

**卸载类即该类的Class对象被GC。**

卸载类需要满足3个要求:

1. 该类的所有的实例对象都已被GC，也就是说堆不存在该类的实例对象。
2. 该类没有在其他任何地方被引用
3. 该类的类加载器的实例已被GC

所以，在JVM生命周期类，由jvm自带的类加载器加载的类是不会被卸载的。但是由我们自定义的类加载器加载的类是可能被卸载的。

只要想通一点就好了，jdk自带的BootstrapClassLoader,PlatformClassLoader,AppClassLoader负责加载jdk提供的类，所以它们(类加载器的实例)肯定不会被回收。而我们自定义的类加载器的实例是可以被回收的，所以使用我们自定义加载器加载的类是可以被卸载掉的。

------

# 四、类加载器    [[2-类加载器]]

## 1、有哪几种类加载器

JVM 中内置了三个重要的 ClassLoader，除了 BootstrapClassLoader 其他类加载器均由 Java 实现且全部继承自`java.lang.ClassLoader`：

1. **BootstrapClassLoader(启动类加载器)** ：最顶层的加载类，由C++实现，负责加载 `%JAVA_HOME%/lib`目录下的jar包和类或者或被 `-Xbootclasspath`参数指定的路径中的所有类。
2. **ExtensionClassLoader(扩展类加载器)** ：主要负责加载目录 `%JRE_HOME%/lib/ext` 目录下的jar包和类，或被 `java.ext.dirs` 系统变量所指定的路径下的jar包。
3. **AppClassLoader(应用程序类加载器)** :面向我们用户的加载器，负责加载当前应用classpath下的所有jar包和类。
4. **用户自定义类加载器：**通过继承 java.lang.ClassLoader类的方式实现。

## 2、什么是双亲委派模型

每一个类都有一个对应它的类加载器。系统中的 ClassLoder 在协同工作的时候会默认使用 **双亲委派模型** 。

- 即在类加载的时候，系统会首先判断当前类是否被加载过。已经被加载的类会直接返回，否则才会尝试加载。

- 加载的时候，首先会把该请求委派该父类加载器的 `loadClass()` 处理，因此所有的请求最终都应该传送到顶层的启动类加载器 `BootstrapClassLoader` 中。

- 当父类加载器无法处理时，才由自己来处理。当父类加载器为null时，会使用启动类加载器 `BootstrapClassLoader` 作为父类加载器。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202403111914891.png)

**概括的说，双亲委派模型就是如果一个类加载器收到了类加载的请求，它首先不会自己去加载这个类，而是把这个请求委派给父类加载器去完成，每一层的类加载器都是如此，这样所有的加载请求都会被传送到顶层的启动类加载器中，只有当父加载无法完成加载请求（它的搜索范围中没找到所需的类）时，子加载器才会尝试去加载类。**

## 3、双亲委派模型的好处
- 双亲委派模型保证了Java程序的稳定运行，**可以避免类的重复加载**（JVM 区分不同类的方式不仅仅根据类名，相同的类文件被不同的类加载器加载产生的是两个不同的类），
- **也保证了 Java 的核心 API 不被篡改**。如果没有使用双亲委派模型，而是每个类加载器加载自己的话就会出现一些问题，比如我们编写一个称为 `java.lang.Object` 类的话，那么程序运行的时候，系统就会出现多个不同的 `Object` 类。


# 参考

https://www.jianshu.com/p/e74fe532e35e



ThinkWon

JavaGuide

https://blog.csdn.net/ThinkWon/article/details/104390752

https://gitee.com/SnailClimb/JavaGuide/blob/master/docs/java/jvm/Java内存区域.md

Java对象头详解

https://gitee.com/SnailClimb/JavaGuide/blob/master/docs/java/jvm/类加载过程.md

https://blog.csdn.net/javazejian/article/details/72828483)