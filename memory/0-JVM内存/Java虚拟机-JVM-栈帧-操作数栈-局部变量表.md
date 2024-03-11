# 参考

https://luolanmeet.github.io/jvm-note/content/part3/%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%89%A7%E8%A1%8C%E5%AD%90%E7%B3%BB%E7%BB%9F/%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E.html

https://mp.weixin.qq.com/s/anzDcV01y13pdFP7U42G-w

https://mp.weixin.qq.com/s/vuDiJlJloYtjLi-Sm9u65A

https://mp.weixin.qq.com/s/gmgaYNM8TsluhXElyeuCYg

https://mp.weixin.qq.com/s/uUpR454JQ2jvMfe5zH_f1A



《The Java Virtual Machine Specification, Java SE 8 Edition》

《Java虚拟机规范》（Java SE 8版）

《深入理解Java虚拟机 JVM高级特性与最佳实践》



# 线索

![image-20210708153515988](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708153516.png)



![image-20210708153551617](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708153551.png)



![image-20210708153609876](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708153609.png)



# JVM基础知识

**一个简单的java程序**

接下来，我们通过一个简单的java程序在JVM层面的执行全过程，方便读者整体地了解上面提到的JVM知识。

```text
public class MathExample {

    private static final int num = 10;

    public static void main(String[] args) {
        MathExample mathExample = new MathExample();
        mathExample.compute();
        System.out.println("finish");
    }

    public int compute(){//一个方法对应一个栈帧内存区域
        int a = 1 ;
        int b = 2 ;
        int c = (a+b) * num ;
        return c;
    }
}
```

该java程序执行全过程如下图所示：

![java程序执行全过程](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708144521.jpg)

## Java从编译到执行

![c385e3518007ed81a7a67f59c5b30b17.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210707174602.webp)





## JVM的跨平台性与语言无关性

![047f66a8b693ab9f59a703d9058d4baa.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210707174638.webp)



## JVM的内存区域

![3f98d9e04d8d049bbf1cad71747d13f9.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210707174830.webp)



> 运行时数据区域：Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域

- 线程私有区域
  - 虚拟机栈
  - 本地方法栈
  - 程序计数器
- 线程共享区
  - 运行时常量池
  - 方法区
  - 堆



## 虚拟机栈

：main -> A -> B -> C, 运行代码，线程1来运行，就会有一个对应的虚拟机栈，同时在执行每个方法的时候都会打包成一个栈帧。

```
/**
 * 虚拟机栈
 */
public class MouthedOrStack {
    public static void main(String[] args) {
        A();
    }

    private static void A() {
        B();
    }

    private static void B() {
        C();
    }

    private static void C() {

    }
}
```

代码的执行过程：如下图所示，执行main()方法的时候入栈，执行A()方法的时候入栈.... 当C()方法执行完了，C()方法出栈，接着B方法运行完了出栈，A方法运行完了出栈，最后main方法执行完了出栈。这个就是Java方法运行对虚拟机栈的一个影响。虚拟机栈就是用来存储线程运行方法中的数据的，每一个方法对应一个栈帧![1f3e4acd10625caaa53a69c59bd35498.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114152.webp):::tips 虚拟机栈是基于线程的，哪怕只有一个main()方法，也是以线程的方式运行的，在线程的生命周期中，参与计算的数据会频繁地入栈和出栈，栈的生命周期和线程一样的 :::

栈的大小限制：`-Xss` 设置线程堆栈大小,不同的操作系统，不同的位数虚拟机栈的大小是不同的。查看jvm的设置

```
-Xsssize
Sets the thread stack size (in bytes). Append the letter k or K to indicate KB, m or M to indicate MB, g or G to indicate GB. The default value depends on the platform:

Linux/ARM (32-bit): 320 KB

Linux/i386 (32-bit): 320 KB

Linux/x64 (64-bit): 1024 KB

OS X (64-bit): 1024 KB

Oracle Solaris/i386 (32-bit): 320 KB

Oracle Solaris/x64 (64-bit): 1024 KB

The following examples set the thread stack size to 1024 KB in different units:

-Xss1m
-Xss1024k
-Xss1048576
This option is equivalent to -XX:ThreadStackSize.
```

虚拟机栈常见的错误：**栈溢出：StackOverflowError  常见的场景：一般无限循环递归会造成这个错误 只有压栈没有弹出栈，虚拟机栈内存也不是无限大的，它是有大小限制的**如下代码：

```
/**
 * 栈异常
 */
public class StackError {
    public static void main(String[] args) {
        A();
    }

    public static void A() {
        A();
    }
}
```

**虚拟机栈存储的主要是栈帧**

**什么是栈帧 (\**\*\*重点\*\**\*)**:::tips 在每个Java方法被调用的时候，都会创建一个栈帧，并入栈。一旦方法完成响应的调用，则出栈 :::**如下图：**![fa1ff847d78125a73a749546b2d188ef.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114203.webp)根据上述的讲解，我们都知道虚拟机栈主要是**存储当前线程运行Java方法中的指令、数据、返回地址**如上图所示，那么每一个方法都是一个栈帧，而栈帧中存储着方法中的变量数据、指令、返回等 栈帧主要包括：局部变量表、操作数栈、动态链接、返回地址。根据如下的代码，来模拟一个方法调用后再栈帧中的处理过程

```
public class OperandStack {
    public int test() {
        int a = 0;
        int b = 1;
        int z = (a + b) * 10;
        return z;
    }

    public static void main(String[] args) {
        OperandStack operandStack = new OperandStack();
        operandStack.test();
    }
}
```

下面我们来模拟一下上述代码中的test方法是如何在虚拟机栈中运行的。如下图是一个，默认状态的虚拟机栈，假设test()在线程1中执行,可以看到局部变量表中有一个this这个是默认的指向的当前的对象。首先，我们需要有一个正确的认知，既然JVM是Java的一个虚拟机，那么JVM就具备一个操作系统所具备的核心功能：CPU + 主内存 + 缓存，那么JVM是一个模拟版的操作系统，JVM执行引擎(CPU) + 栈、堆等(主内存) + 操作数栈(缓存)，首先要知道一个操作系统是如何执行的，比如执行1+1计算，那么这个计算是在CPU中执行，直接结果放到缓存中的，那么JVM也是同样的原理，JVM是通过**执行引擎**计算，将计算结果存入**操作数栈**中。理解这个原理我们就可以很轻松的理解虚拟机栈的运行过程。![1068bf931447d65432675b137aa6c092.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114210.webp)我们都知道JVM是处理class文件中的指令，那么我们需要把上述的Java源代码，编译成class文件，然后通过`javap -c`指令反汇编，来查看class文件中的指令 如下代码就是编译后的class字节码文件

```
public class course01.OperandStack {
  public course01.OperandStack();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public int test();
    Code:
       0: iconst_0
       1: istore_1
       2: iconst_1
       3: istore_2
       4: iload_1
       5: iload_2
       6: iadd
       7: bipush        10
       9: imul
      10: istore_3
      11: iload_3
      12: ireturn

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class course01/OperandStack
       3: dup
       4: invokespecial #3                  // Method "<init>":()V
       7: astore_1
       8: aload_1
       9: invokevirtual #4                  // Method test:()I
      12: pop
      13: return
}
```

这里面涉及到了一些指令，这些指令不需要死记硬背，而是需要去理解，可以根据我提供的链接，直接查找这个指令的意思即可。`首先执行：int a = 0;`在字节码的指令中首先看到的是：`iconst_0` 这个指令是什么意思呢？直接从上述我提供的链接中查找， 意思就是将一个常量0加载到操作数栈，哦原来这个意思就是将int a = 0的常量值放到操作数栈中，这时候线程1中的虚拟机栈就变成了如下：操作数栈中多了一个常量0![916521896a28fb02572306af72db8553.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114219.webp)我们继续往下走，一般指令都是按照顺序执行的，指令是不会跳跃执行，所以我们继续顺着指令，下一个指令执行的是：`istore_1` 查找istore指令是什么意思：讲一个数值从操作数栈存储到局部变量表中，哦原来是这个意思，就是从操作数栈的栈顶取值然后放到局部变量表中，不知道大家有没有注意一个问题`istore_1` 为什么是1而不是0呢？这是因为局部变量表中在0的位置默认有一个this指向当前的对象。(理解每一个步骤及参数的意义是很重要的) 这是JVM执行`istore_1`这个指令，此时虚拟机栈中的情况：局部变量表的1的位置多了一个数值0的常量，此时操作数栈是没有数据的。因为操作数栈的数据出栈存入到局部变量表中了。![5892325c35e11711d2d2f9243892d002.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114226.webp)那么剩下的两个指令执行，我相信就不用在细说了吧  `2: iconst_1` 将常量1存入操作数栈中`3: istore_2` 将操作数栈的数据存入到局部变量表中 此时，虚拟机栈中的情况: 局部变量表中存储着：this、0、1，操作数栈是空的，我们需要理解操作数栈其实就是缓存，主要用来临时存储计算结果的我们不可能在缓存中长久保存数据。（如果还不理解建议学习一下操作系统的基础）![bdf57838d741431856915a9688c44a34.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114232.webp)OK，继续执行指令：iload指令,这个指令是将一个局部变量加载到操作数栈`iload_1` ：将局部变量表中1的位置存储的数据，加载到操作数栈中`iload_2` ：将局部变量表中2的位置存储的数据，加载到操作数栈中 思考：为什么又要加载到操作数栈呢？例如CPU要计算数据，需要从缓存中拿取数据计算，然后将计算结果存入到缓存中。JVM的操作也是同样的原理 此时，虚拟机栈中的运行情况：![d4e421617dc0f65cddcaca0fc141efbe.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114243.webp)继续执行指令：`iadd` : 算法指令 用于对两个操作数栈上的数值进行某种特定的运算，并把结果重新存入到操作数栈顶 此时虚拟机栈的运行情况：首先从操作数栈取出两个数据，由执行引擎进行计算：`(a + b)` 得到的结果1存入到操作数栈的栈顶![d5540fd9eaff6f60c2aa4f55ed95e9b9.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114251.webp)继续执行指令：`bipush     10` ：把一个数值推送到操作数栈栈顶，这里其实执行的代码就相当于：`(a+b)*10` bipush指令直接将数值10推送到操作数栈的栈顶中。![4190028f6c6b8a0e8a1ab836e687bfc8.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114300.webp)然后执行指令：`imul` ：运算指令，对操作数栈中的两个数据进行乘法运算 那么，此时虚拟机栈中的运行情况：10和1进行乘法运算，得到结果10存入到操作数栈中![fe7d45d8ce348935c06b8d00a532858a.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114319.webp)继续执行指令：`istore_3` 将操作数栈存储到局部变量表中，此时虚拟机栈的运行情况如下：![38fd924c137eed7f27f8b985be6ae097.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114325.webp)继续执行指令：`iload_3` ：将局部变量表3的数据，加载到操作数栈中`ireturn` ：将操作数栈中的数据取出，然后压人调用者的栈帧的操作数栈中 此时test()方法执行完毕，会从线程1 的虚拟机栈中出栈，释放 那么此时虚拟机栈的运行情况：此时虚拟机栈中，只剩下main栈帧，而test()方法已经执行完毕了出栈了，需要注意的是main栈帧的操作数栈中有一个常量10这个常量10就是test栈帧执行`ireturn` 指令，压入到main栈帧的操作数栈中的。![c15f306036e971d365f97216e1b60ad0.webp](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114337.webp)main方法执行完毕从虚拟机栈中出栈，OK那么此时，我们的整个代码就执行结束了。其实整个过程就是虚拟机栈的执行的过程，相信大家都理解了虚拟机栈的作用以及运行过程了，嗯....可以吊打面试官了。

- 局部变量表：变量和引用变量 :::tips 局部变量表，用于存放局部变量就是方法中的变量，首先它是一个32位的长度，主要存放Java的八大基础数据类型，如果是64位的就使用高低位占用两个也可以存放下，如果是局部的一些对象，只需要存放它的一个引用地址即可。默认会有一个this当前类的对象的引用 :::
- 操作数栈

操作数栈存放Java方法的操作数的，它就是一个栈结构先进后出。操作数栈就是用来操作，操作的元素可以是任意的Java数据类型，所以当一个方法刚刚开始的时候，这个方法的操作数栈就是空的。操作数栈本质上是JVM执行引擎的一个工作区，也就是所方法在执行，才会对操作数栈进行操作，如果代码不执行，操作数栈其实就是空的。一般操作系统：需要有这些东西CPU + 主内存 + 缓存 :::tips JVM是一个模拟版的操作系统，JVM执行引擎(CPU) + 栈、堆等(主内存) + 操作数栈(缓存) ::: 指令：都是有执行引擎来处理的

```
ICONST_0(iconst_<n>) ： 将一个常量0(n)压入操作数栈
ISTORE 1 (istore_<n>)：表示将操作数栈存入到局部变量表 下标为1(n)的位置
ILOAD 1 :加载存储指令 将局部变量下标为1的值 加载到操作数栈
ILOAD 2 :加载存储指令 将局部变量下标为2的值 加载到操作数栈
IADD : 算法指令 两条数据从操作数栈出栈相加(执行引擎进行计算) ，运算后的结果入到操作数栈(why?) 而执行引擎相当于CPU不做数据的存储，操作数栈相当于缓存，可以存储中间数据
BIPUSH 10 : 将10常量压入操作数栈
IMUL : 算法指令 乘法。 操作数栈出栈，执行引擎计算得到的结果，存入操作数栈
ISTORE 3 ：操作数栈出栈，存入到局部变量表下标为3的位置
ILOAD 3 : 将局部变量下标为3的值 加载到操作数栈
IRETURN : 方法的返回指令： 因为执行引擎都是处理操作数栈中的数据
```

- 动态链接

Java语言的特性多态，具体的会在后面的章节中单独讲解。

- 完成出口 返回地址

正常返回(调用程序计数器中的地址作为返回)、异常的话(通过异常处理器表<非栈帧中的>来确定)





### 虚拟机栈和栈帧

Hotspot JVM 是一个基于栈的虚拟机，每个线程都有一个虚拟机栈用来存储栈帧，每次方法调用都伴随着栈帧的创建、销毁。Java 虚拟机栈的释义如图所示

![image-20210708145349990](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145350.png)

当线程请求分配的栈容量超过 Java 虚拟机栈允许的最大容量时，Java 虚拟机将会抛出 StackOverflowError 异常，可以用 JVM 命令行参数 -Xss 来指定线程栈的大小，比如 -Xss:256k 用于将栈的大小设置为 256KB。

每个线程都拥有自己的 Java 虚拟机栈，一个多线程的应用会拥有多个 Java 虚拟机栈，每个栈拥有自己的栈帧。

![image-20210708145429474](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145429.png)



栈帧是用于支持虚拟机进行方法调用和方法执行的数据结构，随着方法调用而创建，随着方法结束而销毁。栈帧的存储空间分配在 Java 虚拟机栈中，每个栈帧拥有自己的局部变量表（LocalVariable）、操作数栈（Operand Stack）和指向常量池的引用，如图所示。

![image-20210708145501261](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145501.png)



### 局部变量表

每个栈帧内部都包含一组称为局部变量表的变量列表，局部变量表的大小在编译期间就已经确定，对应 class 文件中方法 Code 属性的 locals 字段，Java 虚拟机会根据 locals 字段来分配方法执行过程中需要分配的最大的局部变量表容量。代码示例如下。

```java
public class T {
    public int addFun(int a, int b) {
        return a+b;
    }
}
```

使用 `javac -g:vars T.java` 进行编译，然后执行 `javap -v -l T.class` 查看字节码

```java
public com.example.demo.test.T();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/example/demo/test/T;

  public int addFun(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: ireturn
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       4     0  this   Lcom/example/demo/test/T;
            0       4     1     a   I
            0       4     2     b   I
```

可以发现，默认情况，JVM 会给我们生成一个默认的无惨构造函数。查看每个方法的 LocalVariableTable (局部变量表) 可知，当一个实例方法（非静态方法）被调用时，第 0 个局部变量是调用这个实例方法的对象的引用，也就是我们所说的 this。



### 操作数栈

每个栈帧内部都包含一个称为操作数栈的后进先出（LIFO）栈，栈的大小同样也是在编译期间确定。Java 虚拟机提供的很多字节码指令用于从局部变量表或者对象实例的字段中复制常量或者变量到操作数栈，也有一些指令用于从操作数栈取走数据、操作数据和把操作结果重新入栈。**在方法调用时，操作数栈也用于准备调用方法的参数和接收方法返回的结果**。

比如 iadd 指令用于将两个 int 型的数值相加，它要求执行之前操作数栈已经存在两个 int 型数值，在 iadd 指令执行时，两个 int 型数值从操作数栈中出栈，相加求和，然后将求和的结果重新入栈。1 + 2 对应的指令执行过程，如图所示。



![image-20210708145535168](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145539.png)



整个 JVM 指令执行的过程就是局部变量表与操作数栈之间不断加载、存储的过程，如图所示。

![image-20210708145602760](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145602.png)



那么，如何计算操作数栈的最大值？操作数栈容量最大值对应方法 Code 属性的 stack，表示当前方法的操作数栈在执行过程中任何时间点的最大深度。调用一个成员方法会将 this 和所有参数入栈，调用完毕 this 和参数都会出栈。如果方法有返回值，会将返回值入栈。

```java
public class T {
    public void demo() {
        addFun(123,456);
    }
    public int addFun(int a, int b) {
        return a+b;
    }
}
```

demo 方法的 stack 等于 3，因为调用 addFun 方法会将 this、123、456 这三个变量压栈到栈上，栈的深度为 3，调用完后全部出栈。

字节码如下所示

```java
public void demo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: bipush        123
         3: sipush        456
         6: invokevirtual #2                  // Method addFun:(II)I
         9: pop
        10: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lcom/example/demo/test/T;

  public int addFun(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: ireturn
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       4     0  this   Lcom/example/demo/test/T;
            0       4     1     a   I
            0       4     2     b   I
```

## 字节码指令

### 加载和存储指令

加载（load）和存储（store）相关的指令是使用得最频繁的指令，分为 load 类、store 类、常量加载这三种。

1）load 类指令是将局部变量表中的变量加载到操作数栈，比如 iload_0 将局部变量表中下标为 0 的 int 型变量加载到操作数栈上，根据不同的数据变量类型还有 lload、fload、dload、aload 这些指令，分别表示加载局部变量表中 long、float、double、引用类型的变量。

2）store 类指令是将栈顶的数据存储到局部变量表中，比如 istore_0 将操作数栈顶的元素存储到局部变量表中下标为 0 的位置，这个位置的元素类型为 int，根据不同的数据变量类型还有 lstore、fstore、dstore、astore 这些指令。

3）常量加载相关的指令，常见的有 const 类、push 类、ldc 类。const、push 类指令是将常量值直接加载到操作数栈顶，比如 iconst_0 是将整数 0 加载到操作数栈上，bipush 100 是将 int 型常量 100 加载到操作数栈上。ldc 指令是从常量池加载对应的常量到操作数栈顶，比如 ldc #10 是将常量池中下标为 10 的常量数据加载到操作数栈上。

为什么同是 int 型常量，加载需要分这么多类型呢？这是为了使字节码更加紧凑，int 型常量值根据值 n 的范围，使用的指令按照如下的规则。

❏ 若 n 在 [-1, 5] 范围内，使用 iconst_n 的方式，操作数和操作码加一起只占一个字节。比如 iconst_2 对应的十六进制为 0x05。-1 比较特殊，对应的指令是 iconst_m1（0x02）。

❏ 若 n 在 [-128, 127] 范围内，使用 bipush n 的方式，操作数和操作码一起只占两个字节。比如 n 值为 100（0x64）时，bipush 100 对应十六进制为 0 x1064。

❏ 若 n 在 [-32768, 32767] 范围内，使用 sipush n 的方式，操作数和操作码一起只占三个字节，比如 n 值为 1024（0x0400）时，对应的字节码为 sipush 1024（0x110400）。

❏ 若 n 在其他范围内，则使用 ldc 的方式，这个范围的整数值被放在常量池中，比如 n 值为 40000 时，40000 被存储到常量池中，加载的指令为 ldc # i, i 为常量池的索引值。完整的加载存储指令见表所示。

![image-20210708145637672](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145637.png)



字节码指令的别名很多是使用简写的方式，比如 ldc 是 loadconstant 的简写，bipush 对应 byte immediate push, sipush 对应 short immediate push。

### 操作数栈指令

常见的操作数栈指令有 **pop、dup** 和 **swap**。

**pop** 指令用于将栈顶的值出栈，一个常见的场景是调用了有返回值的方法，但是没有使用这个返回值，比如下面的 demo 方法。

```java
public class T {
    public void demo() {
        addFun(123,"456");
    }
    public String addFun(int a, String b) {
        return a+b;
    }
}
```

demo 方法对应的字节码如下所示

```java
0: aload_0
1: bipush        123
3: ldc           #2                  // String 456
5: invokevirtual #3                  // Method addFun:(ILjava/lang/String;)Ljava/lang/String;
8: pop
9: return
```

第 8 行有一个 pop 指令用于弹出调用 addFun 方法的返回值。

**dup** 指令用来复制栈顶的元素并压入栈顶，后面讲到创建对象的时候会用到 dup 指令。

swap 用于交换栈顶的两个元素，如图所示。

![image-20210708145752024](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145752.png)



还有几个稍微复杂一点的栈操作指令：dup_x1、dup2_x1 和 dup2_x2。下面以 dup_x1 为例来讲解。dup_x1 是复制操作数栈栈顶的值，并插入栈顶以下 2 个值，看起来很绕，把它拆开来看其实分为了五步，如图所示。

![image-20210708145812824](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145812.png)



```java
v1 = stack.pop(); // 弹出栈顶的元素，记为v1
v2 = stack.pop(); // 再次弹出栈顶的元素，记为v2
state.push(v1);   // 将v1 入栈
state.push(v2);   // 将v2 入栈
state.push(v1);   // 再次将v1 入栈
```

接下来看一个 dup_x1 指令的实际例子，代码如下。

```java
class Hello {
    private int id;
    public int incAndGetId() {
        return ++id;
    }
}
```

incAndGetId 方法对应的字节码如下。

```java
Code:
  stack=3, locals=1, args_size=1
     0: aload_0
     1: dup
     2: getfield      #2                  // Field id:I
     5: iconst_1
     6: iadd
     7: dup_x1
     8: putfield      #2                  // Field id:I
    11: ireturn
  LocalVariableTable:
    Start  Length  Slot  Name   Signature
        0      12     0  this   Lcom/example/demo/test/Hello;
```

假如 id 的初始值为 42，调用 incAndGetId 方法执行过程中操作数栈的变化如图所示。

![image-20210708145833783](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145833.png)



第 0 行：aload_0 将 this 加载到操作数栈上。

第 1 行：dup 指令将复制栈顶的 this，现在操作数栈上有两个 this，栈上的元素是 [this, this]。

第 2 行：getfield #2 指令将 42 加载到栈上，同时将一个 this 出栈，栈上的元素变为 [this, 42]。

第 5 行：iconst_1 将常量 1 加载到栈上，栈中元素变为 [this, 42, 1]。

第 6 行：iadd 将栈顶的两个值出栈相加，并将结果 43 放回栈上，现在栈中的元素是 [this, 43]。

第 7 行：dup_x1 将栈顶的元素 43 插入 this 之下，栈中元素变为 [43, this, 43]。

第 8 行：putfield #2 将栈顶的两个元素 this 和 43 出栈，现在栈中元素只剩下栈顶的 [43]，最后的 ireturn 指令将栈顶的 43 出栈返回。完整的操作数栈指令介绍如表所示。

![image-20210708145859204](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145859.png)



### 运算和类型转换指令

Java 中有加减乘除等相关的语法，针对字节码也有对应的运算指令，如表所示。

![image-20210708145917431](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145917.png)



### 控制转移指令

![image-20210708145938901](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708145938.png)





以下面代码中的 isPositive 方法为例，它的作用是判断一个整数是否为正数。

```java
public int isPositive(int n) {
    if (n > 0 ) {
        return 1;
    } else {
        return 0;
    }
}
```

对应的字节码如下所示

```java
Code:
  stack=1, locals=2, args_size=2
     0: iload_1
     1: ifle          6
     4: iconst_1
     5: ireturn
     6: iconst_0
     7: ireturn
  LocalVariableTable:
    Start  Length  Slot  Name   Signature
        0       8     0  this   Lcom/example/demo/test/T;
        0       8     1     n   I
  StackMapTable: number_of_entries = 1
    frame_type = 6 /* same */
```

第 0 行：将局部变量表中下标为 1 的整形变量加载到操作数栈上，也就是加载参数 n

第 1 行：ifle 指令的作用是将操作数栈顶元素出栈跟 0 进行比较，如果小于等于 0 则跳转到特定的字节码处，如果大于 0 则继续执行接下来的字节码。如果栈顶元素小于等于 0，则跳转到第 6 行。

第 4 行：把常量 1 加载到操作数栈上

第 5 行：将栈顶的整数出栈，方法调用结束

第 6 行：把常量 0 加载到操作数栈上

第 7 行：将栈顶的整数出栈，方法调用结束

### for 语句的字节码原理

以 sum 相加求和的例子来看 for 循环的实现细节

```java
public class T {
    public int sum(int[] numbers) {
        int sum = 0;
        for (int i = 0; i < numbers.length; i++) {
            sum = sum + i;
        }
        return sum;
    }
}
```

字节码如下

```java
Code:
  stack=3, locals=4, args_size=2
     /* 将常量0入栈,栈结构[0]*/
     0: iconst_0
     /* 栈顶元素出栈并存储到下标为2的局部变量sum中,栈结构[]*/
     1: istore_2
     /* 将常量0入栈,栈结构[0]*/
     2: iconst_0
     /* 栈顶元素出栈并存储到下标为3的局部变量i中,栈结构[]*/
     3: istore_3
     /* 下标为3的局部变量i入栈,栈结构[i] */
     4: iload_3
     /* 下标为1的局部变量数组numbers入栈,栈结构[numbers,i] */
     5: aload_1
     /* 数组出栈，获取数组长度存储到栈顶，假设数组长度为n,栈结构[n,i] */
     6: arraylength
     /* 栈顶元素出栈n，栈顶元素出栈i。若 i >= n则跳转到22行，否则继续执行 */
     7: if_icmpge     22
     /* 下标为2的局部变量sum入栈,栈结构[sum] */        
    10: iload_2
     /* 下标为1的局部变量数组numbers入栈,栈结构[numbers,sum] */
    11: aload_1
    /* 下标为3的局部变量数组i入栈,栈结构[i,numbers,sum] */
    12: iload_3
    /* i出栈、numbers出栈，把下标为i的数组元素加载到操作数栈上，假设number[i] = x, 栈结构[x,sum]*/
    13: iaload
    /* x出栈，sum出栈，将x与sum相加的结果Y加载到操作数栈上，栈结构[Y] */
    14: iadd
    /* Y出栈存储到本地变量表中下标为2的sum中 */
    15: istore_2
    /* iinc是直接对局部变量进行自增 */
    /* 这里是对局部变量下标为3的变量i进行自增操作并将结果存储到局部变量表里面 */
    16: iinc          3, 1
    /* 跳转到第四行 */    
    19: goto          4
    /* 下标为2的局部变量sum入栈 */
    22: iload_2
    /* 将栈顶的整数出栈，方法调用结束 */
    23: ireturn
  LocalVariableTable:
    Start  Length  Slot  Name   Signature
        4      18     3     i   I
        0      24     0  this   Lcom/example/demo/test/T;
        0      24     1 numbers   [I
        2      22     2   sum   I
  StackMapTable: number_of_entries = 2
    frame_type = 253 /* append */
      offset_delta = 4
      locals = [ int, int ]
    frame_type = 250 /* chop */
      offset_delta = 17
```

### i++ 与 ++i 原理

#### i++ 原理

```java
public class T {
    public static void main(String[] args) {
        int a = 10;
        int b = a++;
        System.out.println(b);
    }
}
```

执行上述代码输出结果是 10，而不是 11。上面代码对应的字节码如下所示

```java
Code:
  stack=2, locals=3, args_size=1
     // 将整数10加载到操作数栈上，操作数栈[10] 
     0: bipush        10
     // 出栈，将结果存储到下标为1的局部变量a中，操作数栈[]    
     2: istore_1
     // 下标为1的局部变量a入栈，操作数栈[x1]，此时的x1=a=10
     3: iload_1
     // iinc indexbyte,constbyte
     // 将整数值constbyte加到下标为indexbyte的int类型的局部变量中。
     // 将本地变量a的值加1，此时a=11
     // 该操作是在本地变量表中执行的，所以此时的操作数栈为[10]，而本地变量表里面的a=11
     4: iinc          1, 1
     // 出栈，将结果加载到下表为2的本地变量b中，所以此时的b=10    
     7: istore_2
     // getstatic、invokevirtual指令后面再分析
     8: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
    11: iload_2
    12: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
    15: return
```

局部变量表如下

| Start | Length | Slot | Name | Signature           |
| ----- | ------ | ---- | ---- | ------------------- |
| 0     | 16     | 0    | args | [Ljava/lang/String; |
| 3     | 13     | 1    | a    | I                   |
| 8     | 8      | 2    | b    | I                   |

#### ++i 原理

```java
public class T {
    public static void main(String[] args) {
        int a = 10;
        int b = ++a;
        System.out.println(b);
    }
}
```

执行上述代码输出结果是 11。上面代码对应的字节码如下所示

```java
Code:
  stack=2, locals=3, args_size=1
     // 将整数10加载到操作数栈上，操作数栈[10]  
     0: bipush        10
     // 出栈，将结果存储到下标为1的局部变量a中，操作数栈[]        
     2: istore_1
     // iinc indexbyte,constbyte
     // 将整数值constbyte加到下标为indexbyte的int类型的局部变量中。
     // 将本地变量a的值加1，此时a=11
     // 该操作是在本地变量表中执行的
     3: iinc          1, 1
     // 下标为1的局部变量a入栈，操作数栈[x1]，此时的x1=a=11    
     6: iload_1
     // 出栈，将结果存储到下标为2的本地变量b中，所以此时的b=11
     7: istore_2
     8: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
    11: iload_2
    12: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
    15: return
```

局部变量表如下

| Start | Length | Slot | Name | Signature           |
| ----- | ------ | ---- | ---- | ------------------- |
| 0     | 16     | 0    | args | [Ljava/lang/String; |
| 3     | 13     | 1    | a    | I                   |
| 8     | 8      | 2    | b    | I                   |

### try-catch 字节码分析

```java
public class T {
    public int fun(int n) {
        try {
            if (n > 10) {
                throw new Exception("n > 10");
            }
        } catch (Exception e) {
            System.out.println("异常");
        }
        return n;
    }
}
```

上述代码对应的字节码

```java
Code:
  stack=3, locals=3, args_size=2
     // 将下标为1的本地变量n入栈，操作数栈[n] 
     0: iload_1
     // 将整数10入栈，操作数栈[10,n] 
     1: bipush        10
     // 10出栈，n出栈，如果 n <= 10 则跳转到 16 行    
     3: if_icmple     16
     //     
     // 创建了一个Exception实例引用,假设为 x,将这个引用压入操作数栈顶， 操作数栈[x] 
     // 此时还没有调用初始化方法，    
     6: new           #2                  // class java/lang/Exception
     // 复制栈顶的元素并压入栈顶, 操作数栈[x,x]  
     9: dup
     // 从常量池加载对应的常量到操作数栈顶,["n > 10",x,x] 
    10: ldc           #3                  // String n > 10
    // 出栈"n > 10"，出栈x，执行构造方法Exception(String message),操作数栈[x]  
    12: invokespecial #4                  // Method java/lang/Exception."<init>":(Ljava/lang/String;)V
    // 将栈顶的异常抛出
    15: athrow
    // 如果不抛出异常就会跳转到28行
    // 如果有异常，则查看Exception table
    // 如果抛出了异常类型为type的异常，就会跳转到target指针表示的字节码处继续执行
    16: goto          28
    // 将接收到的异常存储到下标为2的局部变量e中    
    19: astore_2
    // 调用System.out获取PrintStream并入栈
    20: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    // 将
    23: ldc           #6                  // String 异常
    25: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    // 将下标为1的本地变量n入栈，操作数栈[n] 
    28: iload_1
    // 将栈顶的整数出栈并返回
    29: ireturn
  Exception table:
     from    to  target type
         0    16    19   Class java/lang/Exception
  LocalVariableTable:
    Start  Length  Slot  Name   Signature
       20       8     2     e   Ljava/lang/Exception;
        0      30     0  this   Lcom/example/demo/test/T;
        0      30     1     n   I
  StackMapTable: number_of_entries = 3
    frame_type = 16 /* same */
    frame_type = 66 /* same_locals_1_stack_item */
      stack = [ class java/lang/Exception ]
    frame_type = 8 /* same */
```

### finally 字节码分析

finally 语句块保证一定会执行，以下面的 fun 方法为例，如果 n=10，但是最终返回的结果还是 10 而不是 10。

```java
public class T {
    public int fun(int n) {
        try {
            return n;
        }  finally {
            n = n+10;
        }
    }
}
```

上述代码对应的字节码如下

```java
Code:
  stack=2, locals=4, args_size=2
     // 将下标为1的局部变量n入栈,操作数栈[n] 
     0: iload_1
     // 出栈并保存到下标为2的局部变量中,操作数栈[] 
     1: istore_2
     // 查看Exception table可以发现，如果这里接收到异常，那么将调转到第9行
     // 将下标为1的局部变量n入栈,操作数栈[n] 
     2: iload_1
     // 将整数10入栈,操作数栈[10,n] 
     3: bipush        10
     // 栈顶出栈两个元素出栈并相加，相加结果入栈,[10+n]    
     5: iadd
     // 栈顶元素出栈保存到下标为1的本地变量n中
     6: istore_1
     // 将下标为2的局部变量入栈，操作数栈[n]  
     7: iload_2
     // 栈顶整形元素出栈并返回
     8: ireturn
     // 运行到这里说明程序抛出了异常
     // 出栈并保存到下标为3的局部变量中,操作数栈[] 
     9: astore_3
     // 将下标为1的局部变量n入栈,操作数栈[n] 
    10: iload_1
    // 将整数10入栈,操作数栈[10,n]
    11: bipush        10
    // 栈顶出栈两个元素出栈并相加，相加结果入栈,[10+n]        
    13: iadd
    // 栈顶元素出栈保存到下标为1的本地变量n中
    14: istore_1
    // 将下标为3的局部变量入栈，操作数栈[n]  
    15: aload_3
    // 这里需要将异常抛出
    16: athrow
  Exception table:
     from    to  target type
         0     2     9   any
  LocalVariableTable:
    Start  Length  Slot  Name   Signature
        0      17     0  this   Lcom/example/demo/test/T;
        0      17     1     n   I
  StackMapTable: number_of_entries = 1
    frame_type = 73 /* same_locals_1_stack_item */
      stack = [ class java/lang/Throwable ]
```

从上面的分析可以知道，在执行 `return n;` 之前，会把 n 存储在一个临时变量里面，假设为 X，然后执行 `n=n+10;`，最后返回的确是临时变量 X 的值。

### 对象相关的字节码指令

```java
public class T {
    public int a = 10;
    public static void main(String[] args) {
        T t = new T();
    }
}
```

对应的字节码如下所示

```java
Code:
  stack=2, locals=2, args_size=1
     0: new           #3                  // class com/example/demo/test/T
     3: dup
     4: invokespecial #4                  // Method "<init>":()V
     7: astore_1
     8: return
  LocalVariableTable:
    Start  Length  Slot  Name   Signature
        0       9     0  args   [Ljava/lang/String;
        8       1     1     t   Lcom/example/demo/test/T;
```

一个对象创建需要三条指令，new、dup、＜init> 方法的 invokespecial 调用。在 JVM 中，类的实例初始化方法是＜init>，调用 new 指令时，只是创建了一个类实例引用，将这个引用压入操作数栈顶，此时还没有调用初始化方法。

＜init> 方法是对象初始化方法，类的**构造方法**、**非静态变量的初始化**、**对象初始化代码块**都会被编译进这个方法中。

使用 invokespecial 调用＜init> 方法后才真正调用了构造器方法，那中间的 dup 指令的作用是什么？

invokespecial 会消耗操作数栈顶的类实例引用，如果想要在 invokespecial 调用以后栈顶还有指向新建类对象实例的引用，就需要在调用 invokespecial 之前复制一份类对象实例的引用，否则调用完＜init> 方法以后，类实例引用出栈以后，就再也找不回刚刚创建的对象引用了。有了栈顶的新建对象的引用，就可以使用 astore 指令将对象引用存储到局部变量表

### synchronized 字节码分析

```java
public class T {
    private int count = 0;
    public void increase() {
        synchronized (this) {
            count++;
        }
    }
}
```

字节码

```java
Code:
  stack=3, locals=3, args_size=1
     // 将this对象引用入栈,操作数栈[this] 
     0: aload_0
     // 使用dup指令复制栈顶元素并入栈,操作数栈[this,this]
     1: dup
     // 栈顶元素出栈并将它存入下标为1的局部变量，现在栈上还剩下一个this对象引用。操作数栈[this]
     // 这里通过一个临时变量来存储this的引用
     2: astore_1
     // 栈顶元素this出栈，monitorenter指令尝试获取this对象的监视器锁，如果成功则继续往下执行，
     // 如果已经有其他线程的线程持有，则进入等待状态。
     // 操作数栈[this]
     3: monitorenter
     // 将this对象引用入栈,操作数栈[this] 
     4: aload_0
     // 使用dup指令复制栈顶元素并入栈,操作数栈[this,this]
     5: dup
     // 出栈，并获取this引用的count字段的值并入栈，
     // 假设count的值为x操作数栈[x,this]
     6: getfield      #2                  // Field count:I
     // 整数1入栈,操作数栈[1,x,this]
     9: iconst_1
     // 栈顶两个元素出栈并相加，相加结果入栈,操作数栈[1+x,this]
    10: iadd
    // 1+x出栈、this出栈，将(x+1)的值赋值给this引用的count字段
    11: putfield      #2                  // Field count:I
    // 将下标为1的局部变量入栈，下标为1的局部变量为this的引用,操作数栈[this]
    14: aload_1
    // 出栈，调用monitorexit释放锁
    15: monitorexit
    // 由Exception table发现，如果这里接收到异常，则跳转到19行，
    // 如果没有异常，则跳转到24行
    16: goto          24
    // 将异常结果保存到下标为2的局部变量中
    19: astore_2
    // 将下标为1的局部变量入栈，下标为1的局部变量为this的引用,操作数栈[this]
    20: aload_1
    // 出栈，调用monitorexit释放锁,操作数栈[this]
    21: monitorexit
    // 将下标为2的局部变量入栈
    22: aload_2
    // 出栈并抛出异常
    23: athrow
    // 
    24: return
  Exception table:
     from    to  target type
         4    16    19   any
        19    22    19   any
  LocalVariableTable:
    Start  Length  Slot  Name   Signature
        0      25     0  this   Lcom/example/demo/test/T;
  StackMapTable: number_of_entries = 2
    frame_type = 255 /* full_frame */
      offset_delta = 19
      locals = [ class com/example/demo/test/T, class java/lang/Object ]
      stack = [ class java/lang/Throwable ]
    frame_type = 250 /* chop */
      offset_delta = 4
```

Java 虚拟机保证一个 monitor 一次最多只能被一个线程占有。monitorenter 和 monitorexit 是两个与监视器相关的字节码指令。当线程执行到 monitorenter 指令时，会尝试获取栈顶对象对应监视器（monitor）的所有权，也就是尝试获取对象的锁。如果此时 monitor 没有其他线程占有，当前线程会成功获取锁，monitor 计数器置为 1。如果当前线程已经拥有了 monitor 的所有权，monitorenter 指令也会顺利执行，monitor 计数器加 1。如果其他线程拥有了 monitor 的所有权，当前线程会阻塞，直到 monitor 计数器变为 0。

当线程执行 monitorexit 时，会将监视器计数器减 1，计时器值等于 0 时，锁被释放，其他等待这个锁的线程可以尝试去获取 monitor 的所有权。

编译器必须保证无论同步代码块中的代码以何种方式结束（正常退出或异常退出），代码中每次调用 monitorenter 必须执行对应的 monitorexit 指令。如果执行了 monitorenter 指令但没有执行 monitorexit 指令，monitor 一直被占有，则其他线程没有办法获取锁。如果执行 monitorexit 的线程原本并没有这个 monitor 的所有权，那 monitorexit 指令在执行时将抛出 IllegalMonitorStateException 异常。

为了保证这一点，编译器会自动生成一个异常处理器，这个异常处理器确保了方法正常退出和异常退出都能正常释放锁。





### 栈操作相关

#### load和store

- load 命令：用于将局部变量表的指定位置的相应类型变量加载到栈顶；
- store命令：用于将栈顶的相应类型数据保入局部变量表的指定位置；

| 变量进栈 | 含义                  | 变量保存 | 含义                          |
| :------- | :-------------------- | :------- | :---------------------------- |
| iload    | 第1个int型变量进栈    | istore   | 栈顶int数值存入第1局部变量    |
| iload_0  | 第1个int型变量进栈    | istore_0 | 栈顶int数值存入第1局部变量    |
| iload_1  | 第2个int型变量进栈    | istore_1 | 栈顶int数值存入第2局部变量    |
| iload_2  | 第3个int型变量进栈    | istore_2 | 栈顶int数值存入第3局部变量    |
| iload_3  | 第4个int型变量进栈    | istore_3 | 栈顶int数值存入第4局部变量    |
|          |                       |          |                               |
| lload    | 第1个long型变量进栈   | lstore   | 栈顶long数值存入第1局部变量   |
| fload    | 第1个float型变量进栈  | fstore   | 栈顶float数值存入第1局部变量  |
| dload    | 第1个double型变量进栈 | dstore   | 栈顶double数值存入第1局部变量 |
| aload    | 第1个ref型变量进栈    | astore   | 栈顶ref对象存入第1局部变量    |

#### const、push和ldc

- const、push：将相应类型的常量放入栈顶
- ldc:则是从常量池中将常量

| 常量进栈    | 含义              |
| :---------- | :---------------- |
| aconst_null | null进栈          |
| iconst_m1   | int型常量-1进栈   |
| iconst_0    | int型常量0进栈    |
| iconst_1    | int型常量1进栈    |
| iconst_2    | int型常量2进栈    |
| iconst_3    | int型常量3进栈    |
| iconst_4    | int型常量4进栈    |
| iconst_5    | int型常量5进栈    |
|             |                   |
| lconst_0    | long型常量0进栈   |
| fconst_0    | float型常量0进栈  |
| dconst_0    | double型常量0进栈 |
|             |                   |
| bipush      | byte型常量进栈    |
| sipush      | short型常量进栈   |

| 常量池操作 | 含义                                                 |
| :--------- | :--------------------------------------------------- |
| ldc        | int、float或String型常量从常量池推送至栈顶           |
| ldc_w      | int、float或String型常量从常量池推送至栈顶（宽索引） |
| ldc2_w     | long或double型常量从常量池推送至栈顶（宽索引）       |

#### pop和dup

- pop用于栈顶数值出栈操作；
- dup用于赋值栈顶的指定个数的数值，并将其压入栈顶指定次数；

| 栈顶操作 | 含义                                    |
| :------- | :-------------------------------------- |
| pop      | 栈顶数值出栈(不能是long/double)         |
| pop2     | 栈顶数值出栈(long/double型1个，其他2个) |
|          |                                         |
| dup      | 复制栈顶数值，并压入栈顶                |
| dup_x1   | 复制栈顶数值，并压入栈顶2次             |
| dup_x2   | 复制栈顶数值，并压入栈顶3次             |
| dup2     | 复制栈顶2个数值，并压入栈顶             |
| dup2_x1  | 复制栈顶2个数值，并压入栈顶2次          |
| dup2_x2  | 复制栈顶2个数值，并压入栈顶3次          |
|          |                                         |
| swap     | 栈顶的两个数值互换，且不能是long/double |

**注意：dup2**对于long、double类型的数据就是一个，对于其他类型的数据，才是真正的两个，这个的2代表的是2个slot的数据。

### 2.2 对象相关

#### 字段调用

| 字段调用  | 含义                             |
| :-------- | :------------------------------- |
| getstatic | 获取类的静态字段，将其值压入栈顶 |
| putstatic | 给类的静态字段赋值               |
| getfield  | 获取对象的字段，将其值压入栈顶   |
| putfield  | 给对象的字段赋值                 |

#### 方法调用

| 方法调用        | 作用               | 解释                                   |
| :-------------- | :----------------- | :------------------------------------- |
| invokevirtual   | 调用实例方法       | 虚方法分派                             |
| invokestatic    | 调用类方法         | static方法                             |
| invokeinterface | 调用接口方法       | 运行时搜索合适方法调用                 |
| invokespecial   | 调用特殊实例方法   | 包括实例初始化方法、父类方法           |
| invokedynamic   | 由用户引导方法决定 | 运行时动态解析出调用点限定符所引用方法 |

#### 方法返回

| 方法返回 | 含义               |
| :------- | :----------------- |
| ireturn  | 当前方法返回int    |
| lreturn  | 当前方法返回long   |
| freturn  | 当前方法返回float  |
| dreturn  | 当前方法返回double |
| areturn  | 当前方法返回ref    |

#### 对象和数组

- 创建类实例： new
- 创建数组：newarray、anewarray、multianewarray
- 数组元素 加载到 操作数栈：xaload (x可为b,c,s,i,l,f,d,a)
- 操作数栈的值 存储到数组元素： xastore (x可为b,c,s,i,l,f,d,a)
- 数组长度：arraylength
- 类实例类型：instanceof、checkcast

### 2.3 运算指令

运算指令是用于对操作数栈上的两个数值进行某种运算，并把结果重新存入到操作栈顶。Java虚拟机只支持整型和浮点型两类数据的运算指令，所有指令如下：

| 运算 | int  | long | float | double |
| :--- | :--- | :--- | :---- | :----- |
| 加法 | iadd | ladd | fadd  | dadd   |
| 减法 | isub | lsub | fsub  | dsub   |
| 乘法 | imul | lmul | fmul  | dmul   |
| 除法 | idiv | ldiv | fdiv  | ddiv   |
| 求余 | irem | lrem | frem  | drem   |
| 取反 | ineg | lneg | fneg  | dneg   |

**其他运算：**

- 位移：ishl,ishr,iushr,lshl,lshr,lushr
- 按位或： ior,lor
- 按位与： iand, land
- 按位异或： ixor, lxor
- 自增：iin
- 比较：dcmpg,dcmpl,fcmpg,fcmpl,lcmp

### 2.4 类型转换

类型转换用于将两种不同类型的数值进行转换。

(1) 对于宽化类型转换(小范围向大范围转换)，无需显式的转换指令，并且是安全的操作。各种范围从小到大依次排序： int, long, float, double。

(2)对于窄化类型转换，必须显式地调用类型转换指令，并且该过程很可能导致精度丢失。转换规则中需要特别注意的是当浮点值为NaN, 则转换结果为int或long的0。虽然窄化运算可能会发生上/下限溢出和精度丢失等情况，但虚拟机规范明确规定窄化转换U不可能导致虚拟机抛出异常。

类型转换指令：`i2b, i2c,f2i`等等。

### 2.5 流程控制

控制指令是指有条件或无条件地修改PC寄存器的值，从而达到控制流程的目标

- 条件分支：ifeq、iflt、ifnull、ifnonnull等
- 复合分支：tableswitch、lookupswitch
- 无条件分支：goto、goto_w、jsr、jsr_w、ret

### 2.6 同步与异常

**异常：**

Java程序显式抛出异常： athrow指令。在Java虚拟机中，处理异常(catch语句)不是由字节码指令来实现，而是采用异常表来完成。

**同步：**

方法级的同步和方法内部分代码的同步，都是依靠管程(Monitor)来实现的。

Java语言使用synchronized语句块，那么Java虚拟机的指令集中通过monitorenter和monitorexit两条指令来完成synchronized的功能。为了保证monitorenter和monitorexit指令一定能成对的调用（不管方法正常结束还是异常结束），编译器会自动生成一个异常处理器，该异常处理器的主要目的是用于执行monitorexit指令。

### 2.7 小结

在基于堆栈的的虚拟机中，指令的主战场便是操作数栈，除了load是从局部变量表加载数据到操作数栈以及store储存数据到局部变量表，其余指令基本都是用于操作数栈的。



# JVM指令分析实例一（常量、局部变量、for循环）

Java虚拟机的指令由一个字节长度的、代表着某种特定操作含义的操作码以及跟随其后的零至多个代表此操作所需参数的操作数所构成。虚拟机中许多指令并不包含操作数，只有一个操作码。

Java虚拟机限制操作码的长度为1个字节，因此最多只能有256个指令。

## 指令格式

以下指令格式，是基于Oracle JDK编译后，通过javap工具生成的指令描述格式。

```
<index> <opcode> [<operand1> [<operand2>...]] [<comment>]
<index>
```

指令操作码在方法字节码指令数组中的索引，也可以认为是相对于方法起始处的字节偏移量。其中，指令数组指方法对应的Code属性的code[]数组，该数组用于存放方法的字节码指令。

该索引可以作为控制转移指令的跳转目标。例如，goto 8指令表示跳转到索引为8的指令上继续执行。

```
<opcode>
```

指令的操作码助记符。例如，iconst_0、istore_1、iload_1和return等。

```
<operandN>
```

指令操作数，一条指令可以有0至多个操作数。例如，iconst_0没有操作数，bipush有1个操作数，iinc有2个操作数。

```
<comment>
```

指令行尾的注释。注释内容通常以//开始。

每一行中，表示运行时常量池索引的操作数前，会有一个井号。在指令后的注释中，会带有对这个操作数的描述，例如：

```
 1: invokespecial #8    // Method java/lang/Object."<init>":()V  
10: ldc2_w        #19   // double 100.0d
```

## 实例分析

以下实例均使用JDK 1.8编译，并使用javap生成字节码指令清单。

### 代码1

```
void spin() {
    int i;
    for (i = 0; i < 100; i++) {
        ; // Loop body is empty
    }
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114636.png)

iinc用于实现局部变量的自增操作。在所有字节码指令中，只有该指令可直接用于操作局部变量。

对于非-1至5的int类型常量（对应指令iconst_N），使用bipush来将单字节常量值推至栈顶。

JVM对int类型提供了比较和跳转相结合的if指令，例如该例子中的if_icmplt指令。而对于long、float和double，则需要先通过各自的cmp比较指令计算出int类型结果，再结合int类型的if指令判断后再进行跳转。

### 代码2

```
void dspin() {
    double i;
    for (i = 0.0; i < 100.0; i++) {
        ; // Loop body is empty
    }
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114636.png)

其中，double类型占用局部变量的2个Slot，局部变量索引号从0开始，因此dstore_1对应的局部变量索引为1和2。

由于iinc只针对int类型进行自增操作，JVM并没有提供相应的指令来操作double类型。因此，需要借助dadd来实现double类型的自增操作。

同样，以if开头的比较跳转指令，都只用于int类型。但JVM另外提供了dcmpg、dcmpl来比较两个double类型数值的大小，然后将比较结果（1,0,-1）压入栈顶。最后，再使用int类型的if判断指令来进行判断跳转。

dcmpg与dcmpl的区别仅在于，当比较的其中一个值为NaN时，dcmpg将1压入栈顶，而dcmpl将-1压入栈顶。

ldc相关指令都是将常量值从常量池中推至栈顶。

### 代码3

```
void sspin() {
    short i;
    for (i = 0; i < 100; i++) {
        ; // Loop body is empty
    }
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114636.png)

short类型同样需要通过多条指令来实现i++操作，对应于索引号为5至9的指令。首先，使用iadd实现2个int类型数值相加，再使用i2s指令将int类型结果强制转换为short类型，最后使用istore_1指令将结果存回局部变量i。

对于byte、char和short类型数据，JVM并未提供像int类型一样丰富的直接操作指令。然而，由于byte、char和short类型数据都可以自动宽化转换为int类型，因此均可通过int类型的指令来操作。唯一额外的代价是要将操作结果截短至它们的有效范围内。



# JVM指令分析实例二（算术运算、常量池、控制结构）

相关实例均使用Oracle JDK 1.8编译，并使用javap生成字节码指令清单。

## 算术运算

Java虚拟机通常基于操作数栈进行算术运算。只有iinc指令例外，它直接对局部变量进行自增操作。

### 实例代码

```
int align2agrain(int i, int grain) {
    return ((i + grain - 1) & ~(grain - 1));
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115022)

以上指令，并没有出现取反的指令操作。因为JVM并没有提供取反指令，而是使用异或指令来实现取反。

对一个数进行取反，相当于该数的二进制每一位与1进行异或操作。由于-1的补码二进制表示为全部都是1，因此对一个数进行取反，也相当于-1与该数进行异或。

-1的原码、反码和补码表示

```
[10000001]原=[11111110]反=[11111111]补
```

异或实现取反

```
~x = -1^x
```

## 访问运行时常量池

### 实例代码

```
void useManyNumeric() {
    int i = 100;
    int j = 1000000;
    long l1 = 1;
    long l2 = 0xffffffff;
    double d = 2.2;
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115029)

ldc、ldc_w：将int、float或String类型常量值从常量池中推送至栈顶。

ldc2_w：将long、double类型常量值从常量池中推送至栈顶。（只有宽索引版本）

其中，ldc_w和ldc2_w属于宽索引指令，即指令对应的（索引值）参数为2个字节。而ldc指令对应的（索引值）参数为1个字节。

当运行时常量池中的常量个数超过256个（1个字节所能代表的数量）时，需要使用支持2个字节索引值的指令ldc_w指令来代替ldc访问常量池。

在局部变量表中，long和double类型的数据占用两个连续的局部变量，并且采用两个局部变量中较小的索引值来定位其数据。
因此，lstore_3、lstore 5、dstore 7 这三个指令实际存入的局部变量索引号分别为3和4、5和6、7和8。（局部变量表的索引值从0开始）

## 控制结构

Java虚拟机会根据数据类型的变化来生成不同的条件跳转语句。

### while实例1

```
void whileInt() {
    int i = 0;
    while (i < 100) {
        i++;
    }
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115036.png)

iinc用于实现局部变量的自增操作。在所有字节码指令中，只有该指令可直接用于操作局部变量。

对于循环的实现，将条件判断放在循环的最前面不是更易于理解，为什么要放在最后面？让我们来看看放在最前面的指令序列：

```
2 iload_1
3 bipush 100
5 if_icmpge 14
8 iinc 1,1
11 goto 2
```

显然，两种实现方式第1次循环都要执行5条执行。但对于后续的循环，前者只需要执行4条指令，而后者则需要执行5条指令。因此，将条件判断放在循环的最后面可以更高效的执行循环。

### while实例2

```
void whileDouble() {
    double i = 0;
    while (i < 100.1) {
        i++;
    }
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115043.png)

由于iinc只针对int类型的局部变量进行自增操作，JVM并没有提供相应的指令来操作double类型。因此，需要借助dadd来实现double类型的自增操作。

同样，对于数值类型，以if开头的比较跳转指令，都只支持int类型（对于非数值类型，if比较跳转指令还支持引用类型数值）。因此，JVM另外提供了dcmpg、dcmpl来比较两个double类型数值的大小，然后将比较结果（1,0,-1）压入栈顶。最后，再使用int类型的if判断指令来进行判断跳转。

dcmpg与dcmpl的区别仅在于，当比较的其中一个值为NaN时，dcmpg将1压入栈顶，而dcmpl将-1压入栈顶。

ldc相关指令都是将常量值从常量池中推至栈顶，前面"访问运行时常量池"一节已经介绍过了。

对于for循环分析，请看第一篇：[JVM指令分析实例一（常量、局部变量、for循环）](https://mp.weixin.qq.com/s?__biz=MzU1MDg5NTExNg==&mid=2247483758&idx=1&sn=8b6e16e01387711a7e676531b7a69c35&scene=21#wechat_redirect)

### if实例1

```
int lessThan100(double d) {
    if (d < 100.0) {
        return 1;
    } else {
        return -1;
    }
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115049)

### if实例2

```
int greaterThan100(double d) {
    if (d > 100.0) {
        return 1;
    } else {
        return -1;
    }
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115054.png)

if实例2与if实例1的差别仅在于比较符号由小于号改为大于号，因此ifge指令也相应的变成ifle指令。

如果细心一点，还会发现一个差异，double比较指令由dcmpg变成了dcmpl。

那么，JVM在什么情况下使用dcmpg，什么情况下又会使用dcmpl呢？为了理解这一点，我们需要先回顾一下浮点数中的NaN值。

Java虚拟机关于浮点数的规范

浮点类型包含float和double类型两种，32位单精度和64位双精度与IEEE 754格式的取值与操作是一致的。

NaN值用于表示某此无效的运算操作，例如0除以0等情况。

只要有操作数是NaN，那么对它进行任何数值比较和等值测试都会返回false。任何数值与NaN进行不等值比较都会返回true。

有了以上知识，我们再回到例子来分析一下。

我们知道，dcmpg与dcmpl的作用都是比较两个double类型数值的大小，并将结果（1,0,-1）压入栈顶。区别仅在于，当比较的其中一个值为NaN时，dcmpg将1压入栈顶，而dcmpl将-1压入栈顶。

对于 if (d < 100.0) {}，隐含了两个条件，一个是d必须小于100.0，另一个是d不能为NaN（如果为NaN会返回false）。因此，NaN属于该条件之外的情况。

当 if (d < 100.0) {} 成立时，执行比较指令之后结果为-1。由于满足该条件时d不能为NaN，显然当d为NaN时比较结果不能为-1。因此比较指令排除dcmpl，只能使用dcmpg指令。

## 维基百科对NaN的定义

NaN（Not a Number，非数）是计算机科学中数值数据类型的一类值，表示未定义或不可表示的值。常在浮点数运算中使用。首次引入NaN的是1985年的IEEE 754浮点数标准。

返回NaN的运算有如下三种：

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708114930.png)



# JVM指令分析实例三（方法调用、类实例）

本篇为《JVM指令分析实例》的第三篇，相关实例均使用Oracle JDK 1.8编译，并使用javap生成字节码指令清单。

前两篇传送门：

[JVM指令分析实例一（常量、局部变量、for循环）](https://mp.weixin.qq.com/s?__biz=MzU1MDg5NTExNg==&mid=2247483758&idx=1&sn=8b6e16e01387711a7e676531b7a69c35&scene=21#wechat_redirect)

[JVM指令分析实例二（算术运算、常量池、控制结构）](https://mp.weixin.qq.com/s?__biz=MzU1MDg5NTExNg==&mid=2247483763&idx=1&sn=75c27dd0197b0acda7c8c6a4f23f6170&scene=21#wechat_redirect)

## 方法参数

方法的局部变量表，索引值从0开始，且小于局部变量表的长度。

对于实例方法，JVM会隐式传递一个指向当前实例的引用（this），作为方法的第0个局部变量。因此，应用程序实际传递的参数是从索引值1开始的。

但是，对于类方法，由于不需要传递实例引用。因此，应用程序实际传递的参数是从索引值0开始的。

### 实例代码

```
int addTwo(int i, int j) {
    return i + j;
}

static int addTwoStatic(int i, int j) {
    return i + j;
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115216)

## 方法调用

invokevirtual

该指令用于调用对象的实例方法，根据对象的实际类型进行分派（虚方法分派）。

指令带有一个表示索引的参数，运行时常量池在该索引处的项为某个方法的符号引用（提供类名称、方法名称及方法描述符信息）。

invokestatic

该指令用于调用类方法（static方法）。

invokespecial

该指令用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法。

### 调用实例方法代码

```
int add12and13() {
    return addTwo(12, 13);
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115409)

具体执行流程如下：

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115417)

### 常量池

invokevirtual的参数为常量池索引值16，对应于常量池的Methodref常量类型，表示一个方法。Methodref结构主要包括两部分，Class(类或接口)和NameAndType(字段或方法)，最终都指向字符串常量类型Utf8。

方法的符号引用最终解析结果为：jvm/specification/se8/chapter3/MethodInvoke.addTwo:(II)I，格式为：类全限定名.方法名:方法描述符。

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115425.png)

### 调用类方法代码

```
int add12and13() {
    return addTwoStatic(12, 13);
}
```

#### 字节码指令序列

![图片](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708115433)

调用类方法与调用实例方法相比，指令序列主要有两点差异：

1. 调用类方法不需要传入实例引用this，因此不需要压入栈。
2. 调用类方法使用指令invokestatic（而不是invokevirtual）。

### invokespecial实例代码

```
class Near {
    int it;
    public int getItNear() {
        return getIt(); // 调用私有方法
    }
    private int getIt() {
        return it;
    }
}

class Far extends Near{
    int getItFar() {
        return super.getItNear(); // 调用父类方法
    }
}
```

#### 字节码指令序列

```
jvm.specification.se8.chapter3.Near()：// 调用Near父类Object的实例初始化方法
0: aload_0
1: invokespecial #10    // Method java/lang/Object."<init>":()V
4: return

public int getItNear()：// 调用getIt()私有方法
0: aload_0
1: invokespecial #18    // Method getIt:()I
4: ireturn

int getItFar()：// 调用父类getItNear()方法
0: aload_0
1: invokespecial #16    // Method jvm/specification/se8/chapter3/Near.getItNear:()I
4: ireturn
```

invokespecial指令用于调用实例初始化方法、私有方法和父类方法。

与普通实例方法类似，使用invokespecial指令调用的方法都需要以this作为首个参数。

## 使用类实例

### 实例代码1

```
Object create() {
    return new Object();
}
```

#### 字节码指令序列

```
java.lang.Object create()：
0: new           #3 // class java/lang/Object. 创建对象，并将其引用值压入栈顶
3: dup              // 复制栈顶引用值，并将复制值压入栈顶
4: invokespecial #8 // Method java/lang/Object."<init>":()V. 调用实例初始化方法
7: areturn          // 从当前方法返回对象引用
```

new指令用于创建一个对象。dup指令用于复制栈顶数值，并将复制值压入栈顶。

之所以需要在创建对象之后，再复制一个引用，是因为invokespecial和areturn各需要1个对象引用值。

注意，new指令执行后，并没有完成一个对象实例创建的全部过程，只有执行和完成了实例初始化方法后，实例才算创建完全。

### 实例代码2

```
public class MyObj {
    int i;
    MyObj example() {
        MyObj o = new MyObj();
        return silly(o);
    }
    MyObj silly(MyObj o) {
        if (o != null) {
            return o;
        } else {
            return o;
        }
    }
}
```

#### 字节码指令序列

```
jvm.specification.se8.chapter3.MyObj example()：
 0: new           #1 // class jvm/specification/se8/chapter3/MyObj. 创建MyObj对象并将引用压入栈顶
 3: dup             // 复制栈顶引用值，并将复制值压入栈顶
 4: invokespecial #18 // Method "<init>":()V. 调用MyObj实例初始化方法
 7: astore_1        // 将栈顶引用值存入第2个局部变量（o）
 8: aload_0         // 将第1个局部变量压入栈顶（this）
 9: aload_1         // 将第2个局部变量压入栈顶（o）
10: invokevirtual #19 // Method silly:(Ljvm/specification/se8/chapter3/MyObj;)Ljvm/specification/se8/chapter3/MyObj; 调用实例方法（silly），并将返回结果压入栈顶
13: areturn         // 从当前方法返回对象引用

jvm.specification.se8.chapter3.MyObj silly(jvm.specification.se8.chapter3.MyOb
j)：
0: aload_1          // 将第2个局部变量压入栈顶（o）
1: ifnull        6  // 如果栈顶数值为null，则跳转到索引号为6的指令继续执行
4: aload_1          // 将第2个局部变量压入栈顶（o）
5: areturn          // 从当前方法返回对象引用
6: aload_1          // 将第2个局部变量压入栈顶（o）
7: areturn          // 从当前方法返回对象引用
```

### 实例代码3

```
public class InstanceFieldGetSet {
    int i;
    void setIt(int value) {
        i = value;
    }
    int getIt() {
        return i;
    }
}
```

#### 字节码指令序列

```
void setIt(int)：
0: aload_0          // 将第1个局部变量this压入栈顶
1: iload_1          // 将第2个局部变量value压入栈顶
2: putfield      #18 // Field i:I. 设置this实例的i字段值为value
5: return

int getIt():
0: aload_0          // 将第1个局部变量this压入栈顶
1: getfield      #18 // Field i:I. 获取this实例的字段i的值，并压入栈顶
4: ireturn          // 从当前方法返回int类型结果
```

类实例的字段使用getfield和putfield指令进行访问。

与方法调用指令的操作数类似，putfield及getfield指令的操作数也不代表该字段在类实例中的偏移量。编译器会为实例的这些字段生成符号引用，并保存在运行时常量池之中。这些运行时常量池项会在执行阶段解析为受引用对象中的真实字段位置。



# JVM指令分析实例四（数组、switch）

本篇为《JVM指令分析实例》的第四篇，相关实例均使用Oracle JDK 1.8编译，并使用javap生成字节码指令清单。

前几篇传送门：

[JVM指令分析实例一（常量、局部变量、for循环）](https://mp.weixin.qq.com/s?__biz=MzU1MDg5NTExNg==&mid=2247483758&idx=1&sn=8b6e16e01387711a7e676531b7a69c35&scene=21#wechat_redirect)

[JVM指令分析实例二（算术运算、常量池、控制结构）](https://mp.weixin.qq.com/s?__biz=MzU1MDg5NTExNg==&mid=2247483763&idx=1&sn=75c27dd0197b0acda7c8c6a4f23f6170&scene=21#wechat_redirect)

[JVM指令分析实例三（方法调用、类实例）](https://mp.weixin.qq.com/s?__biz=MzU1MDg5NTExNg==&mid=2247483769&idx=1&sn=e66bdde78157aa88dcdef17219965064&scene=21#wechat_redirect)

## 数组

### 一维原始类型数组

```
void createBuffer() {
    int buffer[];
    int bufsz = 100;
    int value = 12;
    buffer = new int[bufsz];
    buffer[10] = value;
    value = buffer[11];
}
```

#### 字节码指令序列

```
void createBuffer()：
 0: bipush        100   // 将单字节int常量值100压入栈顶
 2: istore_2            // 将栈顶int类型数值100存入第3个局部变量. bufsz = 100
 3: bipush        12    // 将单字节int常量值12压入栈顶
 5: istore_3            // 将栈顶int类型数值12存入第4个局部变量. value = 12
 6: iload_2             // 将第3个int类型局部变量压入栈顶
 7: newarray       int  // 创建int类型数组，并将数组引用值压入栈顶. new int[bufsz]
 9: astore_1            // 将栈顶引用类型值存入第2个局部变量. buffer = new int[bufsz]
10: aload_1             // 将第2个引用类型局部变量压入栈顶
11: bipush        10    // 将单字节int常量10压入栈顶
13: iload_3             // 将第4个int类型局部变量压入栈顶
14: iastore             // 将栈顶int类型数值存入数组的指定索引位置. buffer[10] = value
15: aload_1             // 将第2个引用类型值压入栈顶
16: bipush        11    // 将单字节int常量值11压入栈顶
18: iaload              // 将int类型数组的指定元素压入栈顶
19: istore_3            // 将栈顶int类型数值存入第4个局部变量
20: return
```

newarray指令

创建一个指定原始类型（如int、float、char等）的数组，并将其引用值压入栈顶。

执行该指令后，将从操作数栈出栈1个参数count，类型为int，表示要创建数组的大小。

iastore指令

从操作数栈读取一个int类型数据并存入指定数组中。

执行该指令后，将从操作数栈出栈3个参数arrayref、index和value，在本例中分别对应于第10、11和13索引位置压入的值。

其中，arrayref是一个引用类型值，指向一个int类型的数组。index和value为int类型，index表示待存入数组位置的索引号，value表示待存入index索引位置的值。

iaload指令

从数组中加载一个int类型数据到操作数栈。

执行该指令后，将从操作数栈出栈2个参数arrayref和index，在本例中分别对应于第15和16索引位置压入的值。

其中，arrayref是一个引用类型值，指向一个int类型的数组。index为int类型，表示待加载数组数据的索引号。

### 一维引用类型数组

```
void createThreadArray() {
    Thread threads[];
    int count = 10;
    threads = new Thread[count];
    threads[0] = new Thread();
}
```

#### 字节码指令序列

```
void createThreadArray()：
 0: bipush        10    // 将单字节int类型值10压入栈顶
 2: istore_2            // 将栈顶int类型值存入第3个局部变量. count = 10
 3: iload_2             // 将第3个int类型局部变量压入栈顶
 4: anewarray     #15   // class java/lang/Thread. 创建Thread类型数组，并将数组引用值压入栈顶. new Thread[count]
 7: astore_1            // 将栈顶引用类型值存入第2个局部变量
 8: aload_1             // 将第2个引用类型局部变量压入栈顶
 9: iconst_0            // 将int类型常量0压入栈顶
10: new           #15   // class java/lang/Thread. 创建Thread对象，并将引用值压入栈顶
13: dup                 // 复制栈顶值并压入栈顶
14: invokespecial #17   // Method java/lang/Thread."<init>":()V. 调用实例初始化方法
17: aastore             // 将栈顶引用类型值存入数组的指定索引位置. threads[0] = new Thread()
18: return
```

anewarray指令

创建一个引用类型（如类、接口、数组）数组，并将其引用值压入栈顶。可用于创建一维引用数组，或者用于创建多维数组的一部分。

执行该指令后，将从操作数栈出栈1个参数count，类型为int，表示要创建数组的大小。

aastore指令

（aastore指令与iastore指令作用类似）

从操作数栈读取一个引用类型数据并存入指定数组中。

执行该指令后，将从操作数栈出栈3个参数arrayref、index和value，在本例中分别对应于第8、9和10索引位置压入的值。

其中，arrayref是一个引用类型值，指向一个引用类型的数组。index为int类型，index表示待存入数组位置的索引号。value为引用类型，表示待存入index索引位置的值。

在运行时，value的实际类型必须与arrayref所代表的数组的组件类型相匹配。

### 多维数组

```
int[][][] create3DArray() {
    int grid[][][];
    grid = new int[10][5][];
    return grid;
}
```

#### 字节码指令序列

```
int[][][] create3DArray()：
0: bipush        10         // 将单字节int类型值10压入栈顶. 第1维
2: iconst_5                 // 将int类型常量5压入栈顶. 第2维
3: multianewarray #16,  2   // class "[[[I". 创建int[][][]类型数组，并将引用值压入栈顶
7: astore_1                 // 将栈顶引用类型值存入第2个局部变量
8: aload_1                  // 将第2个引用类型局部变量压入栈顶
9: areturn                  // 从当前方法返回栈顶引用类型值
```

multianewarray指令

创建指定类型和指定维度的多维数组（执行该指令时，操作数栈中必须包含各维度的长度值），并将其引用值压入栈顶。可以用于创建所有类型的多维数组。

对于本实例，数组类型为[[[I，即#16对应的常量池中的符号引用。数组维度为2，两个维度的长度值分别为10和5。虽然int[][][]为3维数组，但由于仅指定了前2个维度的长度值，因此指令对应的维度值为2。

如果指定了第3个维度的长度值，那么在iconst_5之后还需要再将1个int类型长度值压入栈。

所有的数组都有一个与之关联的长度属性，可通过arraylength指令访问。

## switch语句

编译器会使用tableswitch和lookupswitch指令来生成switch语句的编译代码。

Java虚拟机的tableswitch和lookupswitch指令都只能支持int类型的条件值。

tableswitch指令可以高效地从索引表中确定case语句块的分支偏移量。

当switch语句中的case分支条件值比较稀疏时，tableswitch指令的空间使用率偏低。这种情况下，可以使用lookupswitch指令来代替。

### tableswitch指令

```
int chooseNear(int i) {
    switch(i) {
        case 0: return 0;
        case 1: return 1;
        case 2: return 2;
        default: return -1;
    }
}
```

#### 字节码指令序列

```
int chooseNear(int)：
 0: iload_1         // 将第2个int类型局部变量压入栈顶
 1: tableswitch   { // 0 to 2
               0: 28    // 如果case条件值为0，则跳转到索引号为28的指令继续执行
               1: 30    // 如果case条件值为1，则跳转到索引号为30的指令继续执行
               2: 32    // 如果case条件值为2，则跳转到索引号为32的指令继续执行
         default: 34    // 否则，则跳转到索引号为34的指令继续执行
    }
28: iconst_0        // 将int类型常量0压入栈顶
29: ireturn         // 从当前方法返回栈顶int类型数值
30: iconst_1        // 将int类型常量1压入栈顶
31: ireturn         // 从当前方法返回栈顶int类型数值
32: iconst_2        // 将int类型常量2压入栈顶
33: ireturn         // 从当前方法返回栈顶int类型数值
34: iconst_m1       // 将int类型常量-1压入栈顶
35: ireturn         // 从当前方法返回栈顶int类型数值
```

tableswitch指令

用于switch条件跳转，case值连续（变长指令）。

根据索引值在跳转表中寻找配对的分支并进行跳转。

指令格式：tableswitch padbytes defaultbytes lowbytes highbytes jumptablebytes

- padbytes：0~3个填充字节，以使得defaultbytes与方法起始地址（方法内第一条指令的操作码所在的地址）之间的距离是4的位数。
- defaultbytes：32位默认跳转地址
- lowbytes：32位低值low
- highbytes：32位高值high
- jumptablebytes：(high-low+1)个32位有符号数值形成的一张零基址跳转表（0-based jump table）

由于采用了索引值定位的方式（可理解为数组随机访问），因此只需要检查索引是否越界，非常高效。

下面结合实例分析一下：

第1条指令的索引号为0，tableswitch指令索引号为1，为了使defaultbytes与方法起始地址之间的距离是4的位数，所以defaultbytes的开始索引号为4。

defaultbytes、lowbytes和highbytes分别占4个字节，总共12个字节。

case高低值分别为2和0，因此jumptablebytes占用(2-0+1)*4=12个字节。

由于defaultbytes的开始索引号为4，defaultbytes~jumptablebytes共占用24个字节，因此紧跟在tableswitch后面的下一条指令的索引号为4+24=28，对应于实例中的指令"28: iconst_0"。

这里顺便提一下，一般情况下，普通的操作数占1个字节，指向常量池的索引值占2个字节（ldc的常量池索引占1个字节，ldc_w、ldc2_w的常量池索引占2个字节）。所以，方法的指令索引号之间有时不是连续的。

### lookupswitch指令

```
int chooseFar(int i) {
    switch(i) {
        case -100: return -1;
        case 0: return 0;
        case 100: return 1;
        default: return -1;
    }
}
```

#### 字节码指令序列

```
int chooseFar(int)：
 0: iload_1
 1: lookupswitch  { // 3
          -100: 36
             0: 38
           100: 40
       default: 42
    }
36: iconst_m1
37: ireturn
38: iconst_0
39: ireturn
40: iconst_1
41: ireturn
42: iconst_m1
43: ireturn
```

lookupswitch指令

用于switch条件跳转，case值不连续（变长指令）。

根据键值（非索引）在跳转表中寻找配对的分支并进行跳转。

指令格式：lookupswitch padbytes defaultbytes npairsbytes matchoffsetbytes

- padbytes：0~3个填充字节，以使得defaultbytes与方法起始地址（方法内第一条指令的操作码所在的地址）之间的距离是4的位数。
- defaultbytes：32位默认跳转地址
- npairsbytes：32位匹配键值对的数量npairs
- matchoffsetbytes：npairs个键值对，每一组键值对都包含了一个int类型值match以及一个有符号32位偏移量offset。

由于case条件值是非连续的，因此无法采用像tableswitch直接定位的方式，必须对每个键值进行比较。然而，JVM规定，lookupswitch的跳转表必须根据键值排序，这样（如采用二分查找）会比线性扫描更有效率。

下面结合实例分析一下：

第1条指令的索引号为0，lookupswitch指令索引号为1，为了使defaultbytes与方法起始地址之间的距离是4的位数，所以defaultbytes的开始索引号为4。

defaultbytes、npairsbytes分别占4个字节，总共8个字节。

case有3个条件，共3个键值对（npairs为3）。由于每个键值对占8个字节（4字节match+4字节offset），因此matchoffsetbytes共占24个字节。

所以，紧跟在lookupswitch后面的下一条指令的索引号为4+8+24=36，对应于实例中的指令"36: iconst_m1"。

------



# JVM指令分析实例五（操作数栈）

本篇为《JVM指令分析实例》的第五篇，相关实例均使用Oracle JDK 1.8编译，并使用javap生成字节码指令清单。

前几篇传送门：

[JVM指令分析实例一（常量、局部变量、for循环）](https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FanzDcV01y13pdFP7U42G-w)

[JVM指令分析实例二（算术运算、常量池、控制结构）](https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FvuDiJlJloYtjLi-Sm9u65A)

[JVM指令分析实例三（方法调用、类实例）](https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FgmgaYNM8TsluhXElyeuCYg)

[JVM指令分析实例四（数组、switch）](https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FuUpR454JQ2jvMfe5zH_f1A)

## 预备知识

### 局部变量表的变量槽（Variable Slot）

局部变量表的容量以变量槽（Variable Slot）为最小单位，**虚拟机规范中并没有明确指明一个Slot应占用的内存空间大小**。

每个Slot能存放一个boolean、byte、char、short、int、float、reference或returnAddress类型的数据。

reference类型表示对一个对象实例的引用，**虚拟机规范没有说明它的长度及结构**。

returnAddress类型目前已经很少见了，它是为字节码指令jsr、jsr_w和ret服务的，指向了一条字节码指令的地址。

对于64位的数据类型（long、double），虚拟机会以高位对齐的方式为其分配两个连续的Slot空间。

## 操作数栈管理指令

### 复制指令实例代码

```
package jvm.specification.se8.chapter3;
public class NextIndex {
	private long index = 0;
	public long nextIndex() {
		return index++;
	}
}
复制代码
```

#### 字节码指令序列

```
public long nextIndex()：
0: aload_0  // 将第1个局部变量this压入栈顶
1: dup      // 复制栈顶this并压入栈顶. 栈底到栈顶：this、this
2: getfield #12 // Field index:J. 获取实例字段index并压入栈顶，消耗栈顶的1个this. 栈底到栈顶：this、index_for_ladd
5: dup2_x1  // 复制栈顶index数值，并插入第1个this下面. 栈底到栈顶：index_for_return、this、index_for_ladd
6: lconst_1 // 将long类型常量1压入栈顶
7: ladd     // 将栈顶的2个long类型数值相加，并将结果压入栈顶. 栈底到栈顶：index_for_lreturn、this、index_for_putfield
8: putfield #12 // Field index:J. 将栈顶数值赋值给实例字段index
11: lreturn

Constant pool:
   #1 = Class              #2             // jvm/specification/se8/chapter3/NextIndex
   #2 = Utf8               jvm/specification/se8/chapter3/NextIndex
   #3 = Class              #4             // java/lang/Object
   #4 = Utf8               java/lang/Object
   #5 = Utf8               index
   #6 = Utf8               J
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Methodref          #3.#11         // java/lang/Object."<init>":()V
  #11 = NameAndType        #7:#8          // "<init>":()V
  #12 = Fieldref           #1.#13         // jvm/specification/se8/chapter3/NextIndex.index:J
  #13 = NameAndType        #5:#6          // index:J
复制代码
```

**dup2_x1指令**

复制栈顶的1个或2个值，并将其插入到栈顶的2个或3个值下面。

在预备知识中，我们对局部变量的Slot做了简单说明。可以简单理解为，long类型和double类型占2个Slot，其他类型占1个Slot。

下面拆解一下指令的定义（借用局部变量的Slot概念来描述，有点不太严谨，但易于理解与记忆。）。

**复制栈顶的1个或2个值**：

1个可以是long类型和double类型，2个是其他类型，共2个Slot。

**插入到栈顶的2个或3个值下面**：

如果复制的是1个值（即栈顶是long或double，共2个Slot），那么插入到栈顶的2个值（栈顶1个long或double，下面1个其他类型，共3个Slot）下面。

如果复制的是2个值（即栈顶是2个其他类型数值，共2个Slot），那么插入到栈顶的3个值（栈顶3个都是其他类型，共3个Slot）下面。

**简单理解，dup2_x1指令的作用就是将栈顶的2个Slot的值复制并插入到栈顶的3个Slot的值下面。**

对于本实例，执行dup2_x1指令之前的栈结构为（栈底到栈顶）：this、index。由于index为long类型，占2个Slot。this为引用类型，占1个Slot。因此，dup2_x1指令将栈顶的2个Slot的index值复制并插入到栈顶的3个Slot的this引用下面。

### 操作数栈之指令系数法

dup总共有6个指令，分别是dup、dup_x1、dup_x2、dup2、dup2_x1和dup2_x2。初看这些指令，容易混淆而难以理解。**经过分类和找规律，可以通过"指令系数法"来理解记忆，非常简单：**

- 不带_x的指令是复制栈顶数据并压入栈顶。包括两个指令，dup和dup2

- 带_x的指令是复制栈顶数据并插入栈顶以下的某个位置。共有4个指令

- dup的系数代表要复制的Slot个数

  。

  - dup开头的指令用于复制1个Slot的数据。例如1个int或1个reference类型数据
  - dup2开头的指令用于复制2个Slot的数据。例如1个long，或2个int，或1个int+1个float类型数据

- 对于带_x的复制插入指令，只要将指令的dup和x的系数相加，结果即为需要插入的位置

  。因此

  - dup_x1插入位置：1+1=2，即栈顶2个Slot下面。
  - dup_x2插入位置：1+2=3，即栈顶3个Slot下面。
  - dup2_x1插入位置：2+1=3，即栈顶3个Slot下面。
  - dup2_x2插入位置：2+2=4，即栈顶4个Slot下面。

操作数栈管理指令共有9个，上面已经介绍了6个。剩下的3个用同样的方法就很容易理解了：

- pop：将栈顶的1个Slot数值出栈。例如1个short类型数值
- pop2：将栈顶的2个Slot数值出栈。例如1个double类型数值，或者2个int类型数值
- swap：交换栈顶的2个Slot数值位置。Java虚拟机没有提供交换两个64位数据类型（long、double）数值的指令。

备注：指令系数法是自己为了方便记忆起的名字

