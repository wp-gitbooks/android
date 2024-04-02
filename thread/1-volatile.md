---
number headings: auto, first-level 1, max 6, 1.1
---



# 1 线索

what 是什么（用途）


例子


特性：  -------------->实现原理

​			可见性

​			部分有序性

​			原子性				long/double    i++				--------------->AtomicInteger、synchronized



使用：

​		double-checked-locking

​		状态标志

​		开销较低的读



# 2 参考
https://www.pdai.tech/md/java/thread/java-thread-x-key-volatile.html

https://bugstack.cn/interview/2020/10/21/%E9%9D%A2%E7%BB%8F%E6%89%8B%E5%86%8C-%E7%AC%AC14%E7%AF%87-volatile-%E6%80%8E%E4%B9%88%E5%AE%9E%E7%8E%B0%E7%9A%84%E5%86%85%E5%AD%98%E5%8F%AF%E8%A7%81-%E6%B2%A1%E6%9C%89-volatile-%E4%B8%80%E5%AE%9A%E4%B8%8D%E5%8F%AF%E8%A7%81%E5%90%97.html

https://my.oschina.net/u/4303594/blog/4319910

# 3 概述

## 3.1 什么是volatile
volatile和synchronized相比，volatile被称为**轻量级锁**，并且能实现部分synchronized的语义。它在处理多线程并发的时候主要保证了**共享资源的可见性**，该功能可以理解为一个线程修改某一个共享变量的时候，另外一个变量可以读到该共享变量的值。



## 3.2 volatile两核心三性质 [[0.1-Java内存模型JMM#Java内存模型]]

两大核心：JMM内存模型（主内存和工作内存）以及happens-before

三条性质：可见性，原子性，有序性


### 3.2.1 volatile性质

1. 保证了不同线程对这个变量进行操作时的**可见性**，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。（实现可见性）
2. **禁止进行指令重排序**。（实现有序性）
3. **只能保证对单次读/写的原子性**。i++ 这种操作不能保证原子性。（不能实现原子性）
4. volatile不会引起上下文的切换和调度



## 3.3 代码例子

### 3.3.1 1可见性案例

```java
public class ApiTest {

    public static void main(String[] args) {
        final VT vt = new VT();

        Thread Thread01 = new Thread(vt);
        Thread Thread02 = new Thread(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException ignore) {
                }
                vt.sign = true;
                System.out.println("vt.sign = true 通知 while (!sign) 结束！");
            }
        });

        Thread01.start();
        Thread02.start();
    }

}

class VT implements Runnable {

    public boolean sign = false;

    public void run() {
        while (!sign) {
        }
        System.out.println("你坏");
    }
}
```

**这段代码**，是两个线程操作一个变量，程序期望当 `sign` 在线程 Thread01 被操作 `vt.sign = true` 时，Thread02 输出 *你坏*。

但实际上这段代码永远不会输出 *你坏*，而是一直处于死循环。这是为什么呢？接下来我们就一步步讲解和验证。

### 3.3.2 2加上volatile关键字
我们把 sign 关键字加上 volatile 描述，如下：

```java
class VT implements Runnable {

    public volatile boolean sign = false;

    public void run() {
        while (!sign) {
        }
        System.out.println("你坏");
    }
}
```

**测试结果**

```java
vt.sign = true 通知 while (!sign) 结束！
你坏

Process finished with exit code 0
```

volatile关键字是Java虚拟机提供的的最轻量级的同步机制，它作为一个修饰符出现，用来修饰变量，但是这里不包括局部变量哦

在添加 volatile 关键字后，程序就符合预期的输出了 *你坏*。从我们对 volatile 的学习认知可以知道。volatile关键字是 JVM 提供的最轻量级的同步机制，用来修饰变量，用来保证变量对所有线程可见性。

正在修饰后可以让字段在线程见可见，那么这个属性被修改值后，可以及时的在另外的线程中做出相应的反应。

# 4 volatile特性（作用）

## 4.1 volatile可见性
对于一个`volatile`的读总能看到最后一次对于这个`volatile`变量的写


### 4.1.1 原理  (volatile的实现原理) [[重排序#内存屏障]]
[[内存屏障]]

> volatile 变量的内存可见性是基于内存屏障(Memory Barrier)实现:

- 内存屏障，又称内存栅栏，是一个 CPU 指令。
- 在程序运行时，为了提高执行性能，编译器和处理器会对指令进行重排序，JMM 为了保证在不同的编译器和 CPU 上有相同的结果，通过插入特定类型的内存屏障来禁止+ 特定类型的编译器重排序和处理器重排序，插入一条内存屏障会告诉编译器和 CPU：不管什么指令都不能和这条 Memory Barrier 指令重排序。

写一段简单的 Java 代码，声明一个 volatile 变量，并赋值。

```java
public class Test {
    private volatile int a;
    public void update() {
        a = 1;
    }
    public static void main(String[] args) {
        Test test = new Test();
        test.update();
    }
}

```


通过 hsdis 和 jitwatch 工具可以得到编译后的汇编代码:

```bash
......
  0x0000000002951563: and    $0xffffffffffffff87,%rdi
  0x0000000002951567: je     0x00000000029515f8
  0x000000000295156d: test   $0x7,%rdi
  0x0000000002951574: jne    0x00000000029515bd
  0x0000000002951576: test   $0x300,%rdi
  0x000000000295157d: jne    0x000000000295159c
  0x000000000295157f: and    $0x37f,%rax
  0x0000000002951586: mov    %rax,%rdi
  0x0000000002951589: or     %r15,%rdi
  0x000000000295158c: lock cmpxchg %rdi,(%rdx)  //在 volatile 修饰的共享变量进行写操作的时候会多出 lock 前缀的指令
  0x0000000002951591: jne    0x0000000002951a15
  0x0000000002951597: jmpq   0x00000000029515f8
  0x000000000295159c: mov    0x8(%rdx),%edi
  0x000000000295159f: shl    $0x3,%rdi
  0x00000000029515a3: mov    0xa8(%rdi),%rdi
  0x00000000029515aa: or     %r15,%rdi
......


```


**lock 前缀的指令**在多核处理器下会引发两件事情:

- 将当前处理器缓存行的数据写回到**系统内存**。
- 写回内存的操作会使在其他 CPU 里缓存了该内存地址的**数据无效**。

为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存(L1，L2 或其他)后再进行操作，但操作完不知道何时会写到内存。

如果对声明了 volatile 的变量进行写操作，JVM 就会向处理器发送一条 lock 前缀的指令，将这个变量所在缓存行的数据写回到系统内存。

为了保证各个处理器的缓存是一致的，实现了缓存一致性协议(MESI)，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。

所有多核处理器下还会完成：当处理器发现本地缓存失效后，就会从内存中重读该变量数据，即可以获取当前最新值。

volatile 变量通过这样的机制就使得每个线程都能获得该变量的最新值。

#### 4.1.1.1 lock 指令

在 Pentium 和早期的 IA-32 处理器中，lock 前缀会使处理器执行当前指令时产生一个 LOCK# 信号，会对总线进行锁定，其它 CPU 对内存的读写请求都会被阻塞，直到锁释放。 后来的处理器，加锁操作是由高速缓存锁代替总线锁来处理。 因为锁总线的开销比较大，锁总线期间其他 CPU 没法访问内存。 这种场景多缓存的数据一致通过缓存一致性协议(MESI)来保证。

#### 4.1.1.2 缓存一致性
缓存是分段(line)的，一个段对应一块存储空间，称之为缓存行，它是 CPU 缓存中可分配的最小存储单元，大小 32 字节、64 字节、128 字节不等，这与 CPU 架构有关，通常来说是 64 字节。 LOCK# 因为锁总线效率太低，因此使用了多组缓存。 为了使其行为看起来如同一组缓存那样。因而设计了 缓存一致性协议。 缓存一致性协议有多种，但是日常处理的大多数计算机设备都属于 " 嗅探(snooping)" 协议。 所有内存的传输都发生在一条共享的总线上，而所有的处理器都能看到这条总线。 缓存本身是独立的，但是内存是共享资源，所有的内存访问都要经过仲裁(同一个指令周期中，只有一个 CPU 缓存可以读写内存)。 CPU 缓存不仅仅在做内存传输的时候才与总线打交道，而是不停在嗅探总线上发生的数据交换，跟踪其他缓存在做什么。 当一个缓存代表它所属的处理器去读写内存时，其它处理器都会得到通知，它们以此来使自己的缓存保持同步。 只要某个处理器写内存，其它处理器马上知道这块内存在它们的缓存段中已经失效。


### 4.1.2 步骤

**步骤 1：修改本地内存，强制刷回主内存。**

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420101050)


**步骤 2：强制让其他线程的工作内存失效过期。**

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420101117)

**步骤 3：其他线程重新从主内存加载最新值。**

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420101138)




## 4.2 volatile原子性

对任意单个`volatile`变量的读/写具有原子性，但对于类似于`i++`这种复合操作不具有原子性。



### 4.2.1 问题1： i++为什么不能保证原子性?

对于原子性，需要强调一点，也是大家容易误解的一点：对volatile变量的单次读/写操作可以保证原子性的，如long和double类型变量，但是并不能保证i++这种操作的原子性，因为**本质上i++是读、写两次操作**。

现在我们就通过下列程序来演示一下这个问题：

```java
public class VolatileTest01 {
    volatile int i;

    public void addI(){
        i++;
    }

    public static void main(String[] args) throws InterruptedException {
        final  VolatileTest01 test01 = new VolatileTest01();
        for (int n = 0; n < 1000; n++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    test01.addI();
                }
            }).start();
        }
        Thread.sleep(10000);//等待10秒，保证上面程序执行完成
        System.out.println(test01.i);
    }
}

  
        @pdai: 代码已经复制到剪贴板
    
```

大家可能会误认为对变量i加上关键字volatile后，这段程序就是线程安全的。大家可以尝试运行上面的程序。下面是我本地运行的结果：981 可能每个人运行的结果不相同。不过应该能看出，volatile是无法保证原子性的(否则结果应该是1000)。原因也很简单，i++其实是一个复合操作，包括三步骤：

- 读取i的值。
- 对i加1。
- 将i的值写回内存。 volatile是无法保证这三个操作是具有原子性的，我们可以通过AtomicInteger或者Synchronized来保证+1操作的原子性。 注：上面几段代码中多处执行了Thread.sleep()方法，目的是为了增加并发问题的产生几率，无其他作用。
- 

### 4.2.2 问题2： 共享的long和double变量的为什么要用volatile?
**java的内存模型只保证了基本变量的读取操作和写入操作都必须是原子操作的** ，但是对于64位存储的long和double类型来说，JVM读操作和写操作是分开的，分解为2个32位的操作，

这样当多个线程读取一个非volatile得long变量时，可能出现读取这个变量一个值的高32位和另一个值的低32位，从而导致数据的错乱。要在线程间共享long与double字段必须在synchronized中操作或是声明为volatile。

**这里使用volatile，保证了long,double的可见性，那么原子性呢？**

**其实volatile也保证变量的读取和写入操作都是原子操作，注意这里提到的原子性只是针对变量的读取和写入，并不包括对变量的复杂操作，比如i++就无法使用volatile来确保这个操作是原子操作**


## 4.3 volatile部分有序性（如何保证有序性） 
volatile关键字对顺序性的保证就比较霸道，**直接禁止JVM和处理器对volatile关键字修饰的指令重新排序**，但是对于volatile前后无依赖关系的指令则可以随便怎么排序。

```
//a、b 是普通变量，flag 是 volatile 变量
int a = 1;            //代码 1
int b = 2;            //代码 2
boolean flag = true;  //代码 3
int a = 3;            //代码 4
int b = 4;            //代码 5
```

PS：因为 flag 变量是使用 volatile 修饰，则在进行指令重排序时，不会把代码 3 放到代码 1 和代码 2 前面，也不会把代码 3 放到代码 4 或者代码 5 后面。但是指令重排时代码 1 和代码 2 顺序、代码 4 和代码 5 的顺序不在禁止重排范围内，比如：代码 2 可能会被移到代码 1 之前。


**内存屏障类型分为四类。**  
[[重排序#内存屏障]]
[[3-final详解#final域重排序规则]]
https://www.cnblogs.com/yaowen/p/11240540.html

在JMM 中把内存屏障分为四类：
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210808115041.png)

StoreLoad Barriers是一个“全能型”的屏障，它同时具有其他3个屏障的效果。现代的多数处理器大多支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中（Buffer Fully Flush）


**1. LoadLoadBarriers**

指令示例：LoadA —> Loadload —> LoadB

此屏障可以保证 LoadB 和后续读指令都可以读到 LoadA 指令加载的数据，即读操作 LoadA 肯定比 LoadB 先执行。

**2. StoreStoreBarriers**

指令示例：StoreA —> StoreStore —> StoreB

此屏障可以保证 StoreB 和后续写指令可以操作 StoreA 指令执行后的数据，即写操作 StoreA 肯定比 StoreB 先执行。

**3. LoadStoreBarriers**

指令示例： LoadA —> LoadStore —> StoreB

此屏障可以保证 StoreB 和后续写指令可以读到 LoadA 指令加载的数据，即读操作 LoadA 肯定比写操作 StoreB 先执行。

**4. StoreLoadBarriers**

指令示例：StoreA —> StoreLoad —> LoadB

此屏障可以保证 LoadB 和后续读指令都可以读到 StoreA 指令执行后的数据，即写操作 StoreA 肯定比读操作 LoadB 先执行。

**实现有序性的原理：**

如果属性使用了 volatile 修饰，在编译的时候会在该属性的前或后插入上面介绍的 4 类内存屏障来禁止指令重排，比如：

- 在 **volatile 写操作的前面** 插入 StoreStoreBarriers 保证 volatile 写操作之前的普通读写操作执行完毕后再执行 volatile 写操作。
- 在 **volatile 写操作的后面** 插入 StoreLoadBarriers 保证 volatile 写操作后的数据刷新到主内存，保证之后的 volatile 读写操作能使用最新数据(主内存)。
- 在 **volatile 读操作** 的后面插入 LoadLoadBarriers 和 LoadStoreBarriers 保证 volatile 读写操作之后的普通读写操作先把线程本地的变量置为无效，再把主内存的共享变量更新到本地内存，之后都使用本地内存变量。

### 4.3.1 volatile 读操作内存屏障：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420101525)

### 4.3.2 volatile 写操作内存屏障：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420101545)



### 4.3.3 重排序规则

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420103234)

1. 如果第一个操作为volatile读，则不管第二个操作是啥，都不能重排序。这个操作确保volatile读之后的操作不会被编译器重排序到volatile读之前，其前面的所有普通写操作都已经刷新到主内存中；
2. 如果第一个操作volatile写，不管第二个操作是volatile读/写，禁止重排序。
3. 如果第二个操作为volatile写时，则不管第一个操作是啥，都不能重排序。这个操作确保volatile写之前的操作不会被编译器重排序到volatile写之后；
4. 如果第二个操作为volatile读时，不管第二个操作是volatile读/写，禁止重排序

volatile的底层实现是通过插入内存屏障，但是对于编译器来说，发现一个最优布置来最小化插入内存屏障的总数几乎是不可能的，所以，JMM采用了保守策略。如下：

> 在每一个volatile读操作后面插入一个LoadLoad屏障，用来禁止处理器把上面的volatile读与后面任意操作重排序 在每一个volatile写操作前面插入一个StoreStore屏障，用来禁止volatile写与前面任意操作重排序 在每一个volatile写操作后面插入一个StoreLoad屏障，用来禁止volatile写与后面可能有的volatile读/写操作重排序 在每一个volatile读操作前面插入一个LoadStore屏障，用来禁止volatile写与后面可能有的volatile读/写操作重排序

保守策略下，volatile的写插入屏障后生成的指令示意图：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420103307)



> Storestore 屏障可以保证在volatile写之前，其前面的所有普通写操作已经对任意处理器可见了，Storestore 屏障将保障上面所有的普通写在volatile写之前刷新到主内存。

这里比较有意思的是， volatite 写后面的 StoreLoad 屏障的作用是避免volatile写与后面可能有的volatile 读／写操作重排序。

因为编译器常常无法准确判断在一个volatile写的后面是否需要插入一个StoreLoad屏障。为保证能正确实现volatile的内存语义，JMM在采取了保守策略，在每个volatile写的后面，或者在每个 volatile读的前面插入一个StoreLoad屏障。

保守策略下，volatile的读插入屏障后生成的指令示意图：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420103328)

上面的内存屏障插入策略非常保守，在实际执行中，只要不改变volatile写-读的内存语义，编译器可根据情况省略不必要的屏障

举个例子：

```
public class Test {    
int a ;    
volatile int v1 = 1;    
volatile int v2 = 2;    
public  void readWrite(){        
int i = v1;//第一个volatile读        
int j = v2;//第二个volatile读        
a = i+j://普通读        
v1 = i+1;//第一个volatile写       
v2 =j+2;//第二个volatile写    
}    
public synchronized void read(){       
if(flag){            
System.out.println("---i = " + i);         
}    
}
}
```

针对readWrite方法，编译器在生成字节码的时候可以做到如下的优化：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420103443)



> 注意：最后一个storeLoad屏障不能省略。因为第二个volatile写之后，方法立即return，此时编译器无法精准判断后面是否会有vaolatile读或者写。


# 5 volatile的应用场景

## 5.1 double-checked-locking

```
/**
 * 单例模式（懒汉式）
 * @date:2020 年 7 月 14 日 上午 9:48:24
 */
public class Singleton {
    public static volatile Singleton instance = null;
    private Singleton() {
    }
    public static Singleton getInstance() {
        if (instance == null) {     //代码 1
            synchronized (instance) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

```



```
memory = allocate();   
//1、分配对象的内存空间ctorInstance(memory);  
//2、初始化对象instance = memory;     
//3、使instance指向对象的内存空间

```



程序为了优化性能,会将2和3进行重排序,此时执行的顺序是1、3、2,在单线程中,对结果是不会有影响的,可是在多线程程序下,问题就暴露出来了。这时候我们回到刚刚的单例模式中,在实例化的过程中，线程B走到步骤1,发现instance不为空,但是有可能因为指令重排了,导致instance还没有完全初始化,程序就出问题了。为了禁止实例化过程中的重排序,我们用volatile对instance修饰。



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210420102935)



## 5.2 状态标志
**状态标志**，比如布尔类型状态标志，作为完成某个重要事件的标识，此标识不能依赖其他任何变量，Demo 代码如下：

```
public class Flag {
    //任务是否完成标志，true：已完成，false：未完成
    volatile boolean finishFlag;

    public void finish() {
        finishFlag = true;
    }

    public void doTask() { 
        while (!finishFlag) { 
            //keep do task
        }
    }
}
```

## 5.3 开销较低的读

```csharp
/**
 * 计数器
 */
public class Counter {
    private volatile int value;
    //读操作无需加锁，减少同步开销提交性能，使用 volatile 修饰保证读操作的可见性，每次都可以读到最新值 
    public int getValue() {
        return value; 
    }
    //写操作使用 synchronized 加锁，保证原子性
    public synchronized int increment() {
        return value++;
    }
}
```


## 5.4 一次性安全发布(one-time safe publication)

缺乏同步会导致无法实现可见性，这使得确定何时写入对象引用而不是原始值变得更加困难。在缺乏同步的情况下，可能会遇到某个对象引用的更新值(由另一个线程写入)和该对象状态的旧值同时存在。(这就是造成著名的双重检查锁定(double-checked-locking)问题的根源，其中对象引用在没有同步的情况下进行读操作，产生的问题是您可能会看到一个更新的引用，但是仍然会通过该引用看到不完全构造的对象)。

```java
public class BackgroundFloobleLoader {
    public volatile Flooble theFlooble;
 
    public void initInBackground() {
        // do lots of stuff
        theFlooble = new Flooble();  // this is the only write to theFlooble
    }
}
 
public class SomeOtherClass {
    public void doWork() {
        while (true) { 
            // do some stuff...
            // use the Flooble, but only if it is ready
            if (floobleLoader.theFlooble != null) 
                doSomething(floobleLoader.theFlooble);
        }
    }
}

```



## 5.5 独立观察(independent observation)
安全使用 volatile 的另一种简单模式是定期 发布 观察结果供程序内部使用。例如，假设有一种环境传感器能够感觉环境温度。一个后台线程可能会每隔几秒读取一次该传感器，并更新包含当前文档的 volatile 变量。然后，其他线程可以读取这个变量，从而随时能够看到最新的温度值。

```java
public class UserManager {
    public volatile String lastUser;
 
    public boolean authenticate(String user, String password) {
        boolean valid = passwordIsValid(user, password);
        if (valid) {
            User u = new User();
            activeUsers.add(u);
            lastUser = user;
        }
        return valid;
    }
}

```



## 5.6 volatile bean 模式

在 volatile bean 模式中，JavaBean 的所有数据成员都是 volatile 类型的，并且 getter 和 setter 方法必须非常普通 —— 除了获取或设置相应的属性外，不能包含任何逻辑。此外，对于对象引用的数据成员，引用的对象必须是有效不可变的。(这将禁止具有数组值的属性，因为当数组引用被声明为 volatile 时，只有引用而不是数组本身具有 volatile 语义)。对于任何 volatile 变量，不变式或约束都不能包含 JavaBean 属性。

```java
@ThreadSafe
public class Person {
    private volatile String firstName;
    private volatile String lastName;
    private volatile int age;
 
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public int getAge() { return age; }
 
    public void setFirstName(String firstName) { 
        this.firstName = firstName;
    }
 
    public void setLastName(String lastName) { 
        this.lastName = lastName;
    }
 
    public void setAge(int age) { 
        this.age = age;
    }
}  
```



# 6 volatile和synchronize的区别
- volatile只能修饰实例变量或者类变量，synchronize只能修饰方法或者语句块
- volatile无法保证原子性，synchronize能够保证原子性
- volatile和synchronize都能保证有序性，只是实现方式不一样
- volatile不会使线程陷入阻塞，synchronize相反
* volatile比synchronized执行成本更低，因为它不会引起线程上下文的切换和调度
* volatile本质是在告诉jvm当前变量在寄存器（工作内存）中的值是不确定的，需要从主存中读取；synchronized则是锁定当前变量，只有当前线程可以访问该变量，其他线程被阻塞住。
* volatile只能用来修饰变量，而synchronized可以用来修饰变量、方法、和类。
* volatile可以实现变量的可见性，禁止重排序和单次读/写的原子性；而synchronized则可以变量的可见性，禁止重排序和原子性。
* volatile不会造成线程的阻塞；synchronized可能会造成线程的阻塞。
* volatile标记的变量不会被编译器优化；synchronized标记的变量可以被编译器优化



# 7 volatile和atomic原子类区别

* volatile变量可以确保先行关系，即写操作会发生在后续的读操作之前
* 但是volatile对复合操作不能保证原子性。例如用volatile修饰i变量，那么i++操作就不是原子性的。
* atomic原子类提供的atomic方法可以让i++这种操作具有原子性，如getAndIncrement()方法会原子性的进行增量操作把当前值加一，其它数据类型和引用变量也可以进行相似操作，但是atomic原子类一次只能操作一个共享变量，不能同时操作多个共享变量



# 8 volatile面试题

https://www.jianshu.com/p/afb88c9044a7





## 8.1 一个 int 变量用 volatile 修饰，多线程去操作 i++，是否线程安全？如何保证 i++ 线程安全？AtomicInteger 的底层实现原理？

> 使用 AtomicInteger 可以使 i++ 线程安全



## 8.2 线程安全的单例模式有哪几种（单例模式，哪些是安全的）





## 8.3 双重校验锁，也就是单例模式实现中，一些设计问题，比如

- 为什么要加锁？
- 为什么不直接给getInstance方法加锁？
- 为什么需要双重判断是否为空？
- 为什么还要加volatile修饰变量？

具体解析如下：[多线程三问—字节真题](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/5nMxSOgxLo7zLinF1EoF2A)



## 8.4 volatile作用，怎么做到可见性和有序性

## 8.5 volatile讲一下，为什么不能保证原子性