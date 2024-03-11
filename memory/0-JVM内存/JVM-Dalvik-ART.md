# JVM

JVM(Java Virtual Machine) 是一种软件实现，执行像物理机程序的机器（即电脑）

![Java虚拟机结构](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210507192359.jpg)

# Dalvik

- Dalvik虚拟机是Google等厂商合做开发的Android移动设备平台的核心组成部分之一
- 它能够支持已转换为.dex（即Dalvik Executable）格式的Java应用程序的运行，.dex格式是专为Dalvik设计的一种压缩格式，适合内存和处理器速度有限的系统。（dx 是一套工具，能够将 Java .class 转换成 .dex 格式.安全
- 一个dex档一般会有多个.class。因为dex有时必须进行最佳化，会使档案大小增长1-4倍，以ODEX结尾。）

![DVM的运行机制](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210507192447.jpg)

# Dalvik 和标准 Java 虚拟机(JVM)的首要差异

- **Dalvik基于寄存器**
- **JVM基于栈**

- 基于寄存器的虚拟机对于更大的程序来讲，在它们编译的时候，花费的时间更短。
- JVM字节码中，**局部变量会被放入局部变量表中，继而被压入堆栈供操做码进行运算**，固然JVM也能够只使用堆栈而不显式地将局部变量存入变量表中。
- Dalvik字节码中，局部变量会被赋给65536个可用的**寄存器**中的任何一个，Dalvik指令直接操做这些寄存器，而不是访问堆栈中的元素。

# DVM如何运行java

- VM字节码由.class文件组成，每一个文件一个class。工具

- JVM在运行的时候为每个类装载字节码。相反的，Dalvik程序只包含一个.dex文件，这个文件包含了程序中全部的类。

- Java编译器建立了JVM字节码以后，Dalvik的dx编译器删除.class文件，从新把它们编译成Dalvik字节码，而后把它们写进一个.dex文件中。这个过程包括翻译、重构、解释程序的基本元素（常量池、类定义、数据段）。

  常量池描述了全部的常量，包括引用、方法名、数值常量等。类定义包括了访问标志、类名等基本信息。数据段中包含各类被VM执行的函数代码以及类和函数的相关信息（例如DVM所须要的寄存器数量、局部变量表、操做数堆栈大小），还有实例变量。

# Dalvik 和 Java SDK的SDK不一样。

- JDK（Java Develop Toolkit），就是针对JAVA语言的SDK。
- sdk 是 software development kit 是Android 软件开发工具



![DVM和JVM](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210507192524.jpg)DVM和JVM



# Dalvik 和 Java 运行环境的区别 　 　

- Dalvik 通过优化，容许在有限的**内存中同时运行多个虚拟机的实例**，而且每个Dalvik 应用做为一个独立的Linux 进程执行。独立的进程能够防止在虚拟机崩溃的时候全部程序都被关闭。
- Dalvik虚拟机在android2.2以后使用JIT （Just-In-Time）技术，与传统JVM的JIT并不彻底相同，　
- Dalvik虚拟机有本身的 bytecode，并不是使用 java bytecode。
- Dalvik主要是完成对象生命周期管理，堆栈管理，线程管理，安全和异常管理，以及垃圾回收等等重要功能。 　　
- Dalvik负责进程隔离和线程管理，每个android应用在底层都会对应一个独立的Dalvik虚拟机实例，其代码在虚拟机的解释下得以执行。 　　
- 不一样于Java虚拟机运行java字节码，Dalvik虚拟机运行的是其专有的文件格式Dex。 　　
- dex文件格式能够减小总体文件尺寸，提升I/O操做的类查找速度。 　　
- odex是为了在运行过程当中进一步提升性能，对dex文件的进一步优化。 　　
- 全部的Android应用的线程都对应一个linux线程，虚拟机于是能够更多的依赖操做系统的线程调度和管理机制。 　　
- 有一个特殊的虚拟机进程Zygote，他是虚拟机实例的孵化器。它在系统启动的时候就会产生，它会完成虚拟机的初始化、库的加载、预制类库和初始化的操做。若是系统须要一个新的虚拟机实例，它会迅速复制自身，以最快的速度提供给系统。对于一些只读的系统库，全部虚拟机实例都和Zygote共享一块内存区域
- 基于寄存器，基于寄存器的虚拟机虽然比基于堆栈的虚拟机在硬件，通用性上要差一些，可是它的代码执行效率去更好
- 每个Android应用都运行在它本身的DVM实例中，每个DVM实例都是一个独立的进程空间。全部的Android应用的线程都对应一个Linux线程，DVM所以能够更多地依赖操做系统的线程调度和管理机制。不一样的应用在不一样的进程空间里运行，不一样的应用都是用不一样的Linux用户来运行以最大程度地保户应用程序的安全性和独立性

# Art虚拟机

Android 4.4发布了一个ART运行时，准备用来替换掉以前一直使用的Dalvik虚拟机

- 即Android Runtime

  ART 的机制与 Dalvik 不一样。在Dalvik下，应用每次运行的时候，**字节码都须要经过即时编译器（just in time ，JIT）转换为机器码，这会拖慢应用的运行效率，而在ART 环境中，应用在第一次安装的时候，字节码就会预先编译成机器码，使其成为真正的本地应用。这个过程叫作预编译（AOT,Ahead-Of-Time）**。这样的话，应用的启动(首次)和执行都会变得更加快速。

## ART有什么优缺点呢？


- 优势：
  - 一、系统性能的显著提高。
  - 二、应用启动更快、运行更快、体验更流畅、触感反馈更及时。
  - 三、更长的电池续航能力。
  - 四、支持更低的硬件。
- 缺点：
  - 1.机器码占用的**存储空间更大**，字节码变为机器码以后，可能会增长10%-20%（不过在应用包中，可执行的代码经常只是一部分。好比最新的 Google+ APK 是 28.3 MB，可是代码只有 6.9 MB。）
  - 2.应用的**安装时间会变长**。

# tips ：

- 如今智能手机大部分均可以让用户选择使用Dalvik仍是ART模式。固然默认仍是使用Dalvik模式。
- 用法：设置-辅助功能-开发者选项（开发人员工具）-选择运行环境（不一样的手机设置的步骤可能不同）


# 详解  [[AndroidGC简史]]


# 面试题
## 讲一讲Android都用过哪些虚拟机？Dalvik虚拟机和ART虚拟机的区别是什么？


## Dalvik虚拟机与JVM有什么区别
Dalvik 基于寄存器，而 JVM 基于栈。基于寄存器的虚拟机对于更大的程序来说，在它们编译的时候，花费的时间更短。

Dalvik执行.dex格式的字节码，而JVM执行.class格式的字节码。

## 每个应用程序对应多少个Dalvik虚拟机
每一个Android应用在底层都会对应一个独立的Dalvik虚拟机实例，其代码在虚拟机的解释下得以执行 ，而所有的Android应用的线程都对应一个Linux线程	
		
## Dalvik基于JVM的改进
几个class变为一个dex，constant pool，省内存

Zygote，copy-on-write shared,省内存，省cpu，省电

基于寄存器的bytecode，省指令，省cpu，省电

Trace-based JIT,省cpu，省电,省内存	

## Android系统是基于Linux内核的，为什么还要用虚拟机？