# 线索
synchronized是什么？
怎么使用?
synchronized原理

# 概述

###  是什么？
JVM基于进入和退出Monitor对象来实现方法同步和代码块同步。代码块同步是使用**monitorenter和monitorexit指令**实现的，monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处。任何对象都有一个monitor与之关联，当且一个monitor被持有后，它将处于锁定状态。

根据虚拟机规范的要求，在执行monitorenter指令时，首先要去尝试获取对象的锁，如果这个对象没被锁定，或者当前线程已经拥有了那个对象的锁，把锁的计数器加1；相应地，在执行monitorexit指令时会将锁计数器减1，当计数器被减到0时，锁就释放了。如果获取对象锁失败了，那当前线程就要阻塞等待，直到对象锁被另一个线程释放为止。

注意两点：
1、synchronized同步快对同一条线程来说是**可重入**的，不会出现自己把自己锁死的问题；
2、同步块在已进入的线程执行完之前，会阻塞后面其他线程的进入。


#### 特性
- **可重入锁**  [[Java中的锁#可重入锁 VS 非可重入锁]]
- **排他锁** 
- **属于 jvm，由 jvm 实现**

 可重入锁实现可重入性原理或机制是：
​      **每一个锁关联一个线程持有者和计数器，当计数器为 0 时表示该锁没有被任何线程持有，那么任何线程都可能获得该锁而调用相应的方法；**

​      **当某一线程请求成功后，JVM 会记下锁的持有线程，并且将计数器置为 1；此时其它线程请求该锁，则必须等待；而该持有锁的线程如果再次请求这个锁，就可以再次拿到这个锁，同时计数器会递增；** 当线程退出同步代码块时，计数器会递减，如果计数器为 0，则释放该锁。


### 解决了什么问题？
解决了多线程访问的三大问题：可见性、原子性、有序性   [[0.1-Java内存模型JMM#Java内存模型]]


# synchronized用法

## 对象锁

## 类锁

## 代码块锁

在Java中，最简单粗暴的同步手段就是synchronized关键字，其同步的三种用法:
①.同步**实例方法**，锁是当前实例对象
②.同步**类方法**，锁是当前类对象
③.同步**代码块**，锁是括号里面的对象

```
public class SynchronizedTest {

    /**
     * 同步实例方法，锁实例对象
     */
    public synchronized void test() {
    }

    /**
     * 同步类方法，锁类对象
     */
    public synchronized static void test1() {
    }

    /**
     * 同步代码块
     */
    public void test2() {
        // 锁类对象
        synchronized (SynchronizedTest.class) {
            // 锁实例对象
            synchronized (this) {

            }
        }
    }
}
```


# synchronized实现原理
### 例子
javap -verbose查看上述示例:

![image-20210420122500808](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420122500.png)

从图中我们可以看出:


### 结论

 #### 同步方法
 方法级同步没有通过字节码指令来控制，它实现在方法调用和返回操作之中。当方法调用时，调用指令会检查方法ACC_SYNCHRONIZED访问标志是否被设置，若设置了则执行线程需要持有管程(Monitor)才能运行方法，当方法完成(无论是否出现异常)时释放管程。

 #### 同步代码块
 synchronized关键字经过编译后，会在同步块的前后分别形成monitorenter和monitorexit两个字节码指令，**每条monitorenter指令都必须执行其对应的monitorexit指令**，为了保证方法异常完成时这两条指令依然能正确执行，编译器会自动产生一个异常处理器，其目的就是用来执行monitorexit指令(图中14-18、24-30为异常流程)。

 具体看下monitorexit指令做了什么，在Hotspot源码中全文搜索monitorenter，在ciTypeFlow.cpp中找到如下:

```

    case Bytecodes::_monitorenter:
    {
      pop_object();
      set_monitor_count(monitor_count() + 1);
      break;
    }
  case Bytecodes::_monitorexit:
    {
      pop_object();
      assert(monitor_count() > 0, "must be a monitor to exit from");
      set_monitor_count(monitor_count() - 1);
      break;
    }
    
    void      pop_object() {
      assert(is_reference(type_at_tos()), "must be reference type");
      pop();
    }
    
    void      pop() {
      debug_only(set_type_at_tos(bottom_type()));
      _stack_size--;
    }
    int         monitor_count() const  { return _monitor_count; }
    void    set_monitor_count(int mc)  { _monitor_count = mc; }

```

从源码中我们发现**当线程获得该对象锁后，计数器就会加一，释放锁就会将计数器减一**。


# synchronized锁
synchronized用的锁是存在Java对象头里的。

## Java对象头
在 JVM 中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充。如下：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420145356.png)



​		

- 对象头

- 实例变量：存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按 4 字节对齐。

- 填充数据：由于虚拟机要求对象起始地址必须是 8 字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这点了解即可

  
  
  Java 头对象，它实现 **synchronized** 的锁对象的基础，**synchronized** 使用的锁对象是存储在 Java 对象头里的，**jvm 中采用 2 个字来存储对象头(如果对象是数组则会分配 3 个字，多出来的 1 个字记录的是数组长度)，其主要结构是由 Mark Word 和 Class Metadata Address 组成，** 其结构说明如下表：

| 虚拟机位数   | 头对象结构                                                            |
| ------------ | --------------------------------------------------------------------- |
| 说明32/64bit | Mark Word：存储对象的hashCode、锁信息或分代年龄或GC标志等信息32/64bit |
| Class Metadata Address             |     类型指针指向对象的类元数据，JVM通过这个指针确定该对象是哪个类的实例。                                                                  |

​       其中 Mark Word 在默认情况下存储着对象的 HashCode、分代年龄、锁标记位等以下是 32 位 JVM 的 Mark Word 默认存储结构

```
锁状态        25bit   4bit  1bit是否是偏向锁  2bit 锁标志位无锁状态    对象HashCode  对象分代年龄  0   01
```

​       由于对象头的信息是与对象自身定义的数据没有关系的额外存储成本，因此考虑到 JVM 的空间效率，Mark Word 被设计成为一个非固定的数据结构，以便存储更多有效的数据，它会根据对象本身的状态复用自己的存储空间，如 32 位 JVM 下，除了上述列出的 Mark Word 默认存储结构外，还有如下可能变化的结构：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420145602.png)

## Monitor
每个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取该对象的监视器才能进入同步块和同步方法，如果没有获取到监视器的线程将会被阻塞在同步块和同步方法的入口处，进入到BLOCKED状态，如图:

![image-20210420123310751](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420123310.png)

Monitor是线程私有的数据结构，每个线程都有一个可用monitor record列表，同时 还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联(对象头的MarkWord中的LockWord指向monitor的起始地址)，同时monitor中有一个owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。其结构如下:

![image-20210420123254227](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420123254.png)



**Owner**：初始时为NULL表示当前没有任何线程拥有该monitor record，当线程成功拥有该锁后保存线程唯一标识，当锁被释放时又设置为NULL
 **EntryQ**：关联一个系统互斥锁（semaphore），阻塞所有试图锁住monitor record失败的线程
 **RcThis**：表示blocked或waiting在该monitor record上的所有线程的个数
 **Nest**：用来实现重入锁的计数
 **HashCode**：保存从对象头拷贝过来的HashCode值（可能还包含GC age）
 **Candidate**：用来避免不必要的阻塞或等待线程唤醒，因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。Candidate只有两种可能的值0表示没有需要唤醒的线程1表示要唤醒一个继任线程来竞争锁

## Mutex Lock
监视器锁（Monitor）本质是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的。每个对象都对应于一个可称为" 互斥锁" 的标记，这个标记用来保证在任一时刻，只能有一个线程访问该对象。

互斥锁：用于保护临界区，确保同一时间只有一个线程访问数据。对共享资源的访问，先对互斥量进行加锁，如果互斥量已经上锁，调用线程会阻塞，直到互斥量被解锁。在完成了对共享资源的访问后，要对互斥量进行解锁。

**mutex的工作方式：**

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420143819.jpg)

- 1) 申请mutex
- 2) 如果成功，则持有该mutex
- 3) 如果失败，则进行spin自旋. spin的过程就是在线等待mutex, 不断发起mutex gets, 直到获得mutex或者达到spin_count限制为止
- 4) 依据工作模式的不同选择yiled还是sleep
- 5) 若达到sleep限制或者被主动唤醒或者完成yield, 则重复1)~4)步，直到获得为止

由于Java的线程是映射到操作系统的原生线程之上的，如果要阻塞或唤醒一条线程，都需要操作系统来帮忙完成，这就需要从用户态转换到核心态中，因此状态转换需要耗费很多的处理器时间。所以synchronized是Java语言中的一个重量级操作。在JDK1.6中，虚拟机进行了一些优化，譬如在通知操作系统阻塞线程之前加入一段自旋等待过程，避免频繁地切入到核心态中：

synchronized与java.util.concurrent包中的ReentrantLock相比，由于JDK1.6中加入了针对锁的优化措施（见后面），使得synchronized与ReentrantLock的性能基本持平。ReentrantLock只是提供了synchronized更丰富的功能，而不一定有更优的性能，所以在synchronized能实现需求的情况下，优先考虑使用synchronized来进行同步。

# 锁优化-JVM 对 Synchronized 的优化

问题点：

解决什么问题？怎么解决的？
https://zhuanlan.zhihu.com/p/29866981
http://cmsblogs.com/?p=2071




https://xie.infoq.cn/article/d6c45db2c68ca2a5de1630fb3?utm_source=related_read_bottom&utm_medium=article

https://blog.csdn.net/javazejian/article/details/72828483


​       锁的状态总共有四种，**无锁状态、偏向锁、轻量级锁和重量级锁**。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁，但是锁的升级是单向的，**也就是说只能从低到高升级，不会出现锁的降级。**



![image-20210717070918726](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210717070918.png)
## 偏向锁、轻量级锁、重量级锁之间转换

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420143943.jpg)

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420143951.jpg)

偏向所锁，轻量级锁都是乐观锁，重量级锁是悲观锁。

- 一个对象刚开始实例化的时候，没有任何线程来访问它的时候。它是可偏向的，意味着，它现在认为只可能有一个线程来访问它，所以当第一个线程来访问它的时候，它会偏向这个线程，此时，对象持有偏向锁。偏向第一个线程，这个线程在修改对象头成为偏向锁的时候使用CAS操作，并将对象头中的ThreadID改成自己的ID，之后再次访问这个对象时，只需要对比ID，不需要再使用CAS在进行操作。
- 一旦有第二个线程访问这个对象，因为偏向锁不会主动释放，所以第二个线程可以看到对象时偏向状态，这时表明在这个对象上已经存在竞争了。检查原来持有该对象锁的线程是否依然存活，如果挂了，则可以将对象变为无锁状态，然后重新偏向新的线程。如果原来的线程依然存活，则马上执行那个线程的操作栈，检查该对象的使用情况，如果仍然需要持有偏向锁，则偏向锁升级为轻量级锁，（**偏向锁就是这个时候升级为轻量级锁的**），此时轻量级锁由原持有偏向锁的线程持有，继续执行其同步代码，而正在竞争的线程会进入自旋等待获得该轻量级锁；如果不存在使用了，则可以将对象回复成无锁状态，然后重新偏向。
- 轻量级锁认为竞争存在，但是竞争的程度很轻，一般两个线程对于同一个锁的操作都会错开，或者说稍微等待一下（自旋），另一个线程就会释放锁。 但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，**轻量级锁膨胀为重量级锁**，重量级锁使除了拥有锁的线程以外的线程都阻塞，防止CPU空转。


## 偏向锁、轻量级锁、重量级锁
Synchronized是通过对象内部的一个叫做监视器锁（monitor）来实现的，监视器锁本质又是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的。而操作系统实现线程之间的切换需要从**用户态转换到核心态，这个成本非常高，状态之间的转换需要相对比较长的时间，这就是为什么Synchronized效率低的原因**。因此，这种依赖于操作系统Mutex Lock所实现的锁我们称之为“重量级锁”。

Java SE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”：锁一共有4种状态，级别**从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态。锁可以升级但不能降级**。

## 偏向锁
HotSpot的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得。偏向锁是为了在只有一个线程执行同步块时提高性能。

**当一个线程访问同步块并获取锁时**，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令（由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的CAS原子指令的性能消耗）。

**偏向锁获取过程：**
- （1）访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01——确认为可偏向状态。
- （2）如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤（5），否则进入步骤（3）。
- （3）如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行（5）；如果竞争失败，执行（4）。
- （4）如果CAS获取偏向锁失败，则表示有竞争（CAS获取偏向锁失败说明至少有过其他线程曾经获得过偏向锁，因为线程不会主动去释放偏向锁）。当到达全局安全点（safepoint）时，会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着（因为可能持有偏向锁的线程已经执行完毕，但是该线程并不会主动去释放偏向锁），如果线程不处于活动状态，则将对象头设置成无锁状态（标志位为“01”），然后重新偏向新的线程；如果线程仍然活着，撤销偏向锁后升级到轻量级锁状态（标志位为“00”），此时轻量级锁由原持有偏向锁的线程持有，继续执行其同步代码，而正在竞争的线程会进入自旋等待获得该轻量级锁。
- （5）执行同步代码。

**偏向锁的释放过程：**

如上步骤（4）。偏向锁使用了一种等到竞争出现才释放偏向锁的机制：偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

**关闭偏向锁：**

偏向锁在Java 6和Java 7里是默认启用的。由于偏向锁是为了在只有一个线程执行同步块时提高性能，如果你确定应用程序里所有的锁通常情况下处于竞争状态，可以通过JVM参数关闭偏向锁：-XX:-UseBiasedLocking=false，那么程序默认会进入轻量级锁状态。



## 轻量级锁

轻量级锁是为了在线程近乎交替执行同步块时提高性能。

**轻量级锁的加锁过程：**

- （1）在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。这时候线程堆栈与对象头的状态如下图所示。

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420143819.jpg)

- （2）拷贝对象头中的Mark Word复制到锁记录中。
- （3）拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock record里的owner指针指向object mark word。如果更新成功，则执行步骤（3），否则执行步骤（4）。
- （4）如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态，这时候线程堆栈与对象头的状态如下图所示。

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420143925.jpg)

- （5）如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，若当前只有一个等待线程，则可通过自旋稍微等待一下，可能另一个线程很快就会释放锁。 但是**当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁膨胀为重量级锁**，重量级锁使除了拥有锁的线程以外的线程都阻塞，防止CPU空转，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。

**轻量级锁的解锁过程：**
- （1）通过CAS操作尝试把线程中复制的Displaced Mark Word对象替换当前的Mark Word。
- （2）如果替换成功，整个同步过程就完成了。
- （3）如果替换失败，说明有其他线程尝试过获取该锁（此时锁已膨胀），那就要在释放锁的同时，唤醒被挂起的线程。

## 重量级锁
如上轻量级锁的加锁过程步骤（5），轻量级锁所适应的场景是线程近乎交替执行同步块的情况，如果存在同一时间访问同一锁的情况，就会导致轻量级锁膨胀为重量级锁。Mark Word的锁标记位更新为10，Mark Word指向互斥量（重量级锁）

Synchronized的重量级锁是通过对象内部的一个叫做监视器锁（monitor）来实现的，监视器锁本质又是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的。而操作系统**实现线程之间的切换需要从用户态转换到核心态**，这个成本非常高，状态之间的转换需要相对比较长的时间，这就是为什么Synchronized效率低的原因。

（具体见前面的mutex lock）



# 面试题

## Synchronized底层原理，java锁机制

https://mp.weixin.qq.com/s/7_BVmQuPYnRbdifZGLiijw



## synchronized 的 4 种状态

- [不可不说的Java“锁”事](https://links.jianshu.com/go?to=https%3A%2F%2Ftech.meituan.com%2F2018%2F11%2F15%2Fjava-lock.html)
- 访问 synchronized 修饰static方法、synchronized(this|object) 是否会冲突受干扰



## synchronized 修饰 static 方法、普通方法、类、方法块区别



##  访问synchronized 修饰 static 方法、synchronized (类.class) 是否会冲突受干扰

https://juejin.cn/post/6844903482114195463#heading-5



## Synchronized底层原理，java锁机制

https://blog.csdn.net/javazejian/article/details/72828483



https://www.jianshu.com/p/5379356c648f



https://zhuanlan.zhihu.com/p/88884729

https://www.itzhai.com/articles/introduction-and-use-of-reentrantlock.html



## 虚拟机如何实现的synchronized？

http://www.infoq.com/cn/articles/java-se-16-synchronized



## 锁。synchronized、volatile、Lock。锁的几种状态。CAS原理



## synchronized和Lock的使用、区别及底层实现；volatile的作用和使用方式；常见的原子类。



## synchronized中的类锁和对象锁互斥么？

### 对象锁

Java的所有对象都含有1个互斥锁，这个锁由JVM自动获取和释放。

线程进入synchronized方法的时候获取该对象的锁，当然如果已经有线程获取了这个对象的锁，那么当前线程会等待；

synchronized方法正常返回或者抛异常而终止，JVM会自动释放对象锁。

这里也体现了用synchronized来加锁的1个好处，方法抛异常的时候，锁仍然可以由JVM来自动释放。

 

### 类锁

对象锁是用来控制实例方法之间的同步，类锁是用来控制静态方法（或静态变量互斥体）之间的同步。

其实类锁只是一个概念上的东西，并不是真实存在的，它只是用来帮助我们理解锁定实例方法和静态方法的区别的。

我们都知道，java类可能会有很多个对象，但是只有1个Class对象，也就是说类的不同实例之间共享该类的Class对象。Class对象其实也仅仅是1个java对象，只不过有点特殊而已。

由于每个java对象都有1个互斥锁，而类的静态方法是需要Class对象。所以所谓的类锁，不过是Class对象的锁而已。获取类的Class对象有好几种，最简单的就是MyClass.class的方式。



### 结论 

不会相互影响：类锁和对象锁不是同1个东西，一个是类的Class对象的锁，一个是类的实例的锁。也就是说：1个线程访问静态synchronized的时候，允许另一个线程访问对象的实例synchronized方法。反过来也是成立的，因为他们需要的锁是不同的


## 一个文件中有100万个整数，由空格分开，在程序中判断用户输入的整数是否在此文件中。说出最优的方法

## [](https://interview-q-a-1gdnkgkla15afdbe-1258598664.tcloudbaseapp.com/interview/阿里巴巴.html#两个进程同时要求写或者读-能不能实现-如何防止进程的同步)两个进程同时要求写或者读，能不能实现？如何防止进程的同步？



## [](https://interview-q-a-1gdnkgkla15afdbe-1258598664.tcloudbaseapp.com/interview/阿里巴巴.html#given-a-string-determine-if-it-is-a-palindrome-回文-如果不清楚-按字面意思脑补下-considering-only-alphanumeric-characters-and-ignoring-cases)Given a string, determine if it is a palindrome（回文，如果不清楚，按字面意思脑补下）, considering only alphanumeric characters and ignoring cases.

For example, "A man, a plan, a canal: Panama" is a palindrome. "race a car" is not a palindrome.

Note: Have you consider that the string might be empty? This is a good question to ask during an interview. For the purpose of this problem, we define empty string as valid palindrome.

```java
public boolean isPalindrome(String palindrome){
		char[] palindromes = palidrome.toCharArray();
 		if(palindromes.lengh == 0){
	 		return true
 		}
 		Arraylist<Char> temp = new Arraylist();
 		for(int i=0;i<palindromes.length;i++){
 		if((palindromes[i]>'a' && palindromes[i]<'z')||palindromes[i]>'A' && palindromes[i]<'Z')){
 		temp.add(palindromes[i].toLowerCase());
 		}
		}
 		for(int i=0;i<temp.size()/2;i++){
 		if(temp.get(i) != temp.get(temp.size()-i)){
 		//
 		return false;
 		}
 		}
 		return true;
}
```

(https://interview-q-a-1gdnkgkla15afdbe-1258598664.tcloudbaseapp.com/interview/阿里巴巴.html#烧一根不均匀的绳-从头烧到尾总共需要1个小时。现在有若干条材质相同的绳子-问如何用烧绳的方法来计时一个小时十五分钟呢)烧一根不均匀的绳，从头烧到尾总共需要1个小时。现在有若干条材质相同的绳子，问如何用烧绳的方法来计时一个小时十五分钟呢

用两根绳子，一个绳子两头烧，一个一头烧。一根烧完后，把另外一根也两头烧。所以是30+45 共75分钟。





# 参考

https://www.pdai.tech/md/java/thread/java-thread-x-key-synchronized.html#%E9%94%81%E7%9A%84%E7%B1%BB%E5%9E%8B
