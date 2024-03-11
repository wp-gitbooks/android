# 线索
![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303201434519.png)

内存管理?  （虚拟机如何使用内存？）

​        内存泄露、oom

​        gc

执行子系统?

程序编译与优化?

高效并发?（如何让java程序有更高的并发性？）

JVM的工作原理?  （结合实践）

java程序是如何运行的?





OSGI 模块化



![图片](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210720110122)







# Java内存





# 结构

JVM 内存结构是指：Java 虚拟机定义了若干种程序运行期间会使用的运行时数据区，其中有一些会随着虚拟机启动而创建，随着虚拟机退出而销毁，另一些则与线程一一对应，随着线程的开始而创建，随着线程的结束而销毁



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210321184548.jpg)



线程：

​	线程共享

​	线程私有

异常：

StackOverflowError：java虚拟机栈、本地方法栈

OutOfMemoryError：java虚拟机栈、本地方法栈、方法区、堆



![image-20210321190505559](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210321190505.png)



![image-20210321190542752](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210321190727.png)

![image-20210321190659436](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210321190659.png)





# 线索

下一步：同类和对象加载过程联系起来，同内存分配和垃圾回收关联起来

![image-20210508105249157](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210508105315.png)



## JVM内存布局和相应控制

![image-20210321184909055](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210321184909.png)

## 线程共享

### Java堆(Heap)
[[对象访问]]
[[Java堆中对象分配、布局和访问的全过程]]

Java 堆是所有线程共享的一块内存区域，它在虚拟机启动时 就会被创建，并且单个 JVM 进程有且仅有一个 Java 堆。Java 堆是用来存放对象实例及数组，也就是说我们代码中通过 new 关键字 new 出来的对象都存放在这里

![Java 堆内存结构](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210321185249.png)

### 方法区（Method Area）

它存储了每个类的结构信息，例如运行时**常量池**、字段、方法数据、构造函数和普通方法的**字节码**内容，还包括一些在类、实例、接口初始化时用到的特殊方法。



#### 常量池

String类的intern()方法





## 线程私有

跟线程同时创建，所以它跟线程有相同的生命周期

### Java 虚拟机栈（JVM Stacks）

每一个方法在执行的同时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息，每一个方法从调用直至执行完成的过程，就对应着一个栈帧在 Java 虚拟机栈中的入栈到出栈的过程



**局部变量**表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、**对象引用**（reference 类型，它不等同于对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的引用指针，也可能指向一个代表对象的句柄或者其他与此对象相关的位置）和 returnAddress 类型（指向了一条字节码指令的地址）。

其中 64 位长度的 long 和 double 类型的数据会占用 2 个局部变量空间（Slot），其余的数据类型只占用 1 个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小



`Java` 虚拟栈中可能出现两种异常：

- `StackOverflowError`：线程请求的栈深度大于虚拟机所允许的深度
- `OutOfMemoryError`：虚拟机栈扩展时无法申请到足够的内存



### 本地方法栈（Native Method Stacks）

本地方法栈（Native Method Stacks）与 Java 虚拟机栈所发挥的作用是非常相似的，其区别不过是 Java 虚拟机栈为虚拟机执行 Java 方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的 Native 方法服务。



### 程序计数器（Program Counter Register）

程序计数器也是线程私有的，它只需要一块较小的内存空间，每条线程都要有一个独立的程序计数器，你可以把它看作当前线程所执行的字节码的**行号指示**器

字节码解释器工作时就是通过改变这个计数器的值来选取**下一条需要执行的字节码指令**，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成



- 如果线程正在执行 `Java` 方法，则计数器记录的是正在执行的虚拟机字节码指令的地址
- 如果执行 `native` 方法，则计数器为空



**程序计数器是唯一一个在Java虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域**。



## 特殊（直接内存）

直接内存(Direct Memory)不是虚拟机运行时数据区的一部分，也不是java虚拟机规范中的内存区域，但也可能导致OutOfMemoryError



NIO：DirectByteBuffer直接操作，避免Java堆和Native堆来回赋值数据



# 面试题

## 常见面试题
https://www.bilibili.com/video/BV1Kp4y1z7L8?p=2

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210731215311.png)


## 运行时数据区域分区，哪些线程私有，哪些线程共享。栈帧的数据结构。方法区存放哪些数据。



## 静态类

- 静态类的静态方法能不能被子类重写（不能,走的invoke-static指令,只有走invoke-virtual指令的才可能实现多态）



## JVM

- JVM的内存模型
- JMM的内存模型
- 为什么操作数栈是Thread-Private



## OOM

https://juejin.cn/post/6948397601515372557



https://juejin.cn/post/6844904163927654408#heading-2



## Dalvik跟JVM的区别 [[JVM-Dalvik-ART]]

https://www.cnblogs.com/yuanqiangfei/p/12252521.html



https://blog.csdn.net/weixin_41101173/article/details/79884683


## String  [[String面试题#new String 创建几个对象？]]