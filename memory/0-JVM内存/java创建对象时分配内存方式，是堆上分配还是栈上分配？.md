# 概述
创建对象的内存是分配在堆上还是栈上面？大部分童鞋的回答是这样的：“肯定分配在堆内存的嘛，栈内存是属于子线程和基本数据类型专用的内存空间，怎么会分配到栈上面呢？”，这个回答嘛，也对，也不对，说他对，没错，确实是堆上分配的，说他不对，是因为得看具体情况，那么接下来就为大家介绍下，**什么是栈上分配，什么是堆上分配**；

首先我们得先了解一个概念，现在java的虚拟机默认使用的都是 oracle公司的hotsport虚拟机，在控制台输入 ：java -version 就会打印出java版本以及虚拟机的信息

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202304061104219.png)

不可否认，大部分的对象创建时都是分配到堆内存里面的，但是呢也有特例，以**hotsport虚拟机为例，hotspot虚拟机在创建对象的时候会先判断你这个创建的对象符不符合栈上分配的条件，如果符合，那么这个对象就会分配到栈上；否则就分配到堆上**；

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202304061104220.png)

# 分配到栈内存的条件

那分配到堆还是栈的这个条件是什么呢？其实就是满不满足逃逸分析的条件了，童鞋们就更懵了：“脑师，啥叫逃逸分析啊？”，刚刚我们说了，创建对象默认都是分配到堆内存里面的，如果分配到栈内存的话，这是一种比较**极端的方式**，一般情况下不这么干，只有极端情况下才会分配到栈内存，这也是hotspot的一种优化技术，所以啊，满足逃逸分析的条件之后，创建的对象就会分配到栈内存了

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202304061104221.png)


# 逃逸分析

逃逸分析其实很好理解，就是当你在方法里 new 出一个对象之后，这个对象能不能逃出当前的方法，如果对象**不能逃出**当前方法，就代表着满足了逃逸分析的《**不可逃逸**》条件；就是说你这个对象除了在当前方法使用，还有没有在别的地方使用，如果有用到，就代表着对象逃出了当前方法（**可逃逸**），其实逃逸分析的条件有2个：

1.  不能逃出当前方法，--**不可逃逸**条件
2.  这个方法执行了上万次，才会执行逃逸分析的逻辑判断

满足了以上2个条件了之后，创建出的对象才会分配到栈内存，否则就会分配到堆内存；

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202304061104222.png)

# 演示

接下来，我们用代码演示一下 ，这里先讲一下，hotspot默认是开启了逃逸分析的；在默认情况下，我们执行下面的代码，并且加上虚拟机参数 ： **-XX:+PrintGC** ，看看虚拟机有没进行垃圾回收

```java

      package com.test;
      import java.io.ObjectInputStream;
      public class Test {
         public static void main(String[] args) throws InterruptedException {
             long l = System.currentTimeMillis();
             for(int i = 50000000; i> 0; i --){
                 // 调用方法后，里面的 obj 没有在别的地方引用，属于不可逃逸
                  newObject();
              }
              System.out.println("一共执行了"+(System.currentTimeMillis()- l) +"ms");
          }
         static void newObject(){
             Object obj = new Object();
          }
      }
  
```

执行结果：

> 一共执行了4ms
> 
> Process finished with exit code 0

由结果我们可以看到，jvm 并没有进行垃圾回收机制，并且五千万次的循环只用了4毫秒；那么接下来我们关掉逃逸分析；在启动的时候加入jvm参数 

```diff
-XX:-DoEscapeAnalysis
```

以上的参数表示关闭逃逸分析功能，默认值是 **-XX:+DoEscapeAnalysis** ， 很好理解吧，+表示启用，-表示禁用，代码还是原来的代码，加上参数后，在看看打印的结果：

> [GC (Allocation Failure)  65536K->552K(251392K), 0.0012776 secs]  
> [GC (Allocation Failure)  66088K->552K(251392K), 0.0009459 secs]  
> [GC (Allocation Failure)  66088K->568K(251392K), 0.0007338 secs]  
> [GC (Allocation Failure)  66104K->528K(316928K), 0.0008565 secs]  
> [GC (Allocation Failure)  131600K->536K(316928K), 0.0011731 secs]  
> [GC (Allocation Failure)  131608K->584K(438272K), 0.0009762 secs]  
> 一共执行了272ms
> 
> Process finished with exit code 0

禁用逃逸分析之后，我们可以看到， 虚拟机一共进行了6次清理，每次清理时工作线程都会暂停一段时间，禁用逃逸分析后，执行时间也由之前的4ms一下就飙到了272ms；

# 完

其实知道了这种原理在我们敲代码的时候并不会用到，只是在面试时会用到，但是却可以帮助我们了解底层的原理机制；对这篇文章，你有什么样的想法呢？欢迎评论！