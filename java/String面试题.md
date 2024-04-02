# 线索
![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210816110547.png)

new String
intern

[[#Triple kill]]


# new String()创建几个对象？

## 常见面试问题

下面代码中创建了几个对象？

```java
new String("abc");
```

答案众说纷纭，有说创建了1个对象，也有说创建了2个对象。答案对，也不对，关键是要学到问题底层的原理。

### 底层原理分析

在上篇文章《面试题系列第1篇：说说\=\=和equals的区别？你的回答可能是错误的》中我们已经提到，String的两种初始化形式是有本质区别的。

```java
String str1 = "abc";  // 在常量池中

String str2 = new String("abc"); // 在堆上
```

当直接赋值时，**字符串“abc”会被存储在常量池中，只有1份，此时的赋值操作等于是创建0个或1个对象**。**如果常量池中已经存在了“abc”，那么不会再创建对象，直接将引用赋值给str1；如果常量池中没有“abc”，那么创建一个对象，并将引用赋值给str1**。

那么，通过new String(“abc”);的形式又是如何呢？答案是1个或2个。

当JVM遇到上述代码时，会先检索常量池中是否存在“abc”，如果不存在“abc”这个字符串，则会先在常量池中创建这个一个字符串。然后再执行new操作，会在堆内存中创建一个存储“abc”的String对象，对象的引用赋值给str2。**此过程创建了2个对象**。

当然，如果检索常量池时发现已经存在了对应的字符串，那么只会在堆内创建一个新的String对象，此过程只创建了1个对象。

在上述过程中**检查常量池是否有相同Unicode的字符串常量时，使用的方法便是String中的intern()方法**。

```java
public native String intern();
```

下面通过一个简单的示意图看一下String在内存中的两种存储模式。 

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210806215111.png)


上面的示意图我们可以看到在堆内创建的String对象的char value[]属性指向了常量池中的char value[]。

还是上面的示例，如果我们通过debug模式也能够看到String的char value[]的引用地址。

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210806215147.png)

图中两个String对象的value值的引用均为{char[3]@1355}，也就是说，虽然是两个对象，但它们的value值均指向常量池中的同一个地址。当然，大家还可以拿一个复杂对象（Person）的字符串属性（name）相同时的debug结果进行比对，结果是一样的。

### 深入问法

如果面试官说程序的代码只有下面一行，那么会创建几个对象？

```javascript
new String("abc");
```

答案是2个？

还真不一定。之所以单独列出这个问题是想提醒大家一点：没有直接的赋值操作（str=“abc”），并不代表常量池中没有“abc”这个字符串。也就是说衡量创建几个对象、常量池中是否有对应的字符串，不仅仅由你是否创建决定，还要看程序启动时其他类中是否包含该字符串。

### 升级加码

以下实例我们暂且不考虑常量池中是否已经存在对应字符串的问题，假设都不存在对应的字符串。

以下代码会创建几个对象：

```java
String str = "abc" + "def";
```

上面的问题涉及到字符串常量重载“+”的问题，当一个字符串由多个字符串常量拼接成一个字符串时，它自己也肯定是字符串常量。字符串常量的“+”号连接Java虚拟机会在程序编译期将其优化为连接后的值。

就上面的示例而言，在编译时已经被合并成“abcdef”字符串，因此，只会创建1个对象。并没有创建临时字符串对象abc和def，这样减轻了垃圾收集器的压力。

我们通过javap查看class文件可以看到如下内容。 

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210806215214.png)

很明显，字节码中只有拼接好的abcdef。

针对上面的问题，我们再次升级一下，下面的代码会创建几个对象？

```javascript
String str = "abc" + new String("def");
```

创建了4个，5个，还是6个对象？

4个对象的说法：常量池中分别有“abc”和“def”，堆中对象new String(“def”)和“abcdef”。

这种说法对吗？不完全对，如果说上述代码创建了几个字符串对象，那么可以说是正确的。但上述的代码Java虚拟机在编译的时候同样会优化，会创建一个StringBuilder来进行字符串的拼接，实际效果类似：

```java
String s = new String("def");
new StringBuilder().append("abc").append(s).toString();
```

很显然，多出了一个StringBuilder对象，那就应该是5个对象。

那么创建6个对象是怎么回事呢？有同学可能会想了，StringBuilder最后toString()之后的“abcdef”难道不在常量池存一份吗？

这个还真没有存，我们来看一下这段代码：

```java
@Test
public void testString3() {
	String s1 = "abc";
	String s2 = new String("def");
	String s3 = s1 + s2;
	String s4 = "abcdef";
	System.out.println(s3==s4); // false
}
```

按照上面的分析，如果s1+s2的结果在常量池中存了一份，那么s3中的value引用应该和s4中value的引用是一样的才对。下面我们看一下debug的效果。

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210806215232.png)

很明显，s3和s4的值相同，但value值的地址并不相同。即便是将s3和s4的位置调整一下，效果也一样。s4很明确是存在于常量池中，那么s3对应的值存储在哪里呢？很显然是在堆对象中。

我们来看一下StringBuilder的toString()方法是如何将拼接的结果转化为字符串的：

```java
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}
```

很显然，在toString方法中又新创建了一个String对象，而该String对象传递数组的构造方法来创建的：

```java
public String(char value[], int offset, int count) 
```

也就是说，String对象的value值直接指向了一个已经存在的数组，而并没有指向常量池中的字符串。

因此，上面的准确回答应该是创建了4个字符串对象和1个StringBuilder对象。

### 小结

我们通过一行创建字符串的代码逐步分析String对象的整个构建及拼接过程，了解了底层实现原理。是不是很有意思？当你掌握了这些底层基本知识，即便面试题的形式如何变化，你必定能一眼识破真相。

# String.intern()使用总结

https://www.yuque.com/renyong-jmovm/ds/ffbz0p

## First Blood

先看下面的代码：


```java
String s = new String("1");
String s1 = s.intern();
System.out.println(s == s1);
```



```java
打印结果为：
false
```


对于`new String("1")`，会生成两个对象，一个是String类型对象，它将存储在**Java Heap**中，另一个是字符串常量对象"1"，它将存储在**字符串常量池**中。

`s.intern()`方法首先会去字符串常量池中查找是否存在字符串常量对象"1"，如果存在则返回该对象的地址，如果不存在则在字符串常量池中生成为一个"1"字符串常量对象，并返回该对象的地址。

如下图：

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210816110516.png)

变量`s`指向的是Stirng类型对象，变量`s1`指向的是"1"字符串常量对象，所以`s == s1`结果为false。


## Double kill

在上面的基础上我们再定义一个s2如下：

```java
String s = new String("1");
String s1 = s.intern();
String s2 = "1";
System.out.println(s == s1);
System.out.println(s1 == s2); // true
```

`s1 == s2`为true，表示变量s2是直接指向的字符串常量，如下图：

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210816110532.png)



## Triple kill

在上面的基础上我们再定义一个t如下：

```java
String s = new String("1");
String t = new String("1");
String s1 = s.intern();
String s2 = "1";
System.out.println(s == s1);
System.out.println(s1 == s2);
System.out.println(s == t);   // false
System.out.println(s.intern() == t.intern());   // true
```


`s == t`为false，这个很明显,变量s和变量t指向的是不同的两个String类型的对象。

`s.intern() == t.intern()`为true，因为intern方法返回的是字符串常量池中的同一个"1"对象，所以为true。

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210816110547.png)

## Ultra kill

在上面的基础上我们再定义一个x和s3如下：


```
String s = new String("1");
String t = new String("1");
String x = new String("1") + new String("1");
String s1 = s.intern();
String s2 = "1";
String s3 = "11";
System.out.println(s == s1);
System.out.println(s1 == s2);
System.out.println(s == t);
System.out.println(s.intern() == t.intern());
System.out.println(x == s3);  // fasle
System.out.println(x.intern() == s3.intern());  // true
```


变量x为两个String类型的对象相加，因为`x != s3`，所以x肯定不是指向的字符串常量，实际上x就是一个String类型的对象，调用`x.intern()`方法将返回"11"对应的字符串常量，所以`x.intern() == s3.intern()`为true。



## Rampage

将上面的代码简化并添加几个变量如下：

```
String x = new String("1") + new String("1");
String x1 = new String("1") + "1";
String x2 = "1" + "1";
String s3 = "11";

System.out.println(x == s3);  // false
System.out.println(x1 == s3);  // false
System.out.println(x2 == s3); // true
```


`x == s3`为false表示x指向String类型对象，s3指向字符串常量;

`x1 == s3`为false表示x1指向String类型对象，s3指向字符串常量;

`x2 == s3`为true表示x2指向字符串常量，s3指向字符串常量;


所以我们可以看到`new String("1") + "1"`返回的String类型的对象。


## 总结

现在我们知道intern方法就是将字符串保存到常量池中，在保存字符串到常量池的过程中会先查看常量池中是否已经存在相等的字符串，如果存在则直接使用该字符串。

所以我们在写业务代码的时候，应该尽量使用字符串常量池中的字符串，比如使用`String s = "1";`比使用`new String("1");`更节省内存。我们也可以使用`String s = 一个String类型的对象.intern();`方法来间接的使用字符串常量，这种做法通常用在你接收到一个String类型的对象而又想节省内存的情况下，当然你完全可以String s = 一个String类型的对象;但是这么用可能会因为变量s的引用而影响String类型对象的垃圾回收。所以我们可以使用intern方法进行优化，但是需要注意的是`intern`能节省内存，但是会影响运行速度，因为该方法需要去常量池中查询是否存在某字符串。



# String相关
https://cloud.tencent.com/developer/article/1686226


## 字符型常量和字符串常量的区别？

1.  **形式上**: 字符常量是单引号引起的一个字符，字符串常量是双引号引起的若干个字符；
  
2.  **含义上**: 字符常量相当于一个整型值( ASCII 值),可以参加表达式运算；字符串常量代表一个地址值(该字符串在内存中存放位置，相当于对象；
  
3.  **占内存大小**：字符常量只占2个字节；字符串常量占若干个字节(至少一个字符结束标志) (注意: char 在Java中占两个字节)。
  

## 什么是字符串常量池？

java中常量池的概念主要有三个：`全局字符串常量池`，`class文件常量池`，`运行时常量池`。我们现在所说的就是`全局字符串常量池`，对这个想弄明白的同学可以看这篇[Java中几种常量池的区分](http://tangxman.github.io/2015/07/27/the-difference-of-java-string-pool/)。

jvm为了提升性能和减少内存开销，避免字符的重复创建，其维护了一块特殊的内存空间，即字符串池，当需要使用字符串时，先去字符串池中查看该字符串是否已经存在，如果存在，则可以直接使用，如果不存在，初始化，并将该字符串放入字符串常量池中。

字符串常量池的位置也是随着jdk版本的不同而位置不同。在jdk6中，常量池的位置在永久代（方法区）中，此时常量池中存储的是**对象**。在jdk7中，常量池的位置在堆中，此时，常量池存储的就是**引用**了。在jdk8中，永久代（方法区）被元空间取代了。

## String str="aaa"与 String str=new String("aaa")一样吗？`new String("aaa");`创建了几个字符串对象?

-   使用`String a = “aaa” ;`，程序运行时会在常量池中查找”aaa”字符串，若没有，会将”aaa”字符串放进常量池，再将其地址赋给a；若有，将找到的”aaa”字符串的地址赋给a。
-   使用String b = new String("aaa");`，程序会在堆内存中开辟一片新空间存放新对象，同时会将”aaa”字符串放入常量池，相当于创建了两个对象，无论常量池中有没有”aaa”字符串，程序都会在堆内存中开辟一片新空间存放新对象。

具体分析，见以下代码：

```
 @Test
    public void test(){
        String s = new String("2");
        s.intern();
        String s2 = "2";
        System.out.println(s == s2);


        String s3 = new String("3") + new String("3");
        s3.intern();
        String s4 = "33";
        System.out.println(s3 == s4);
    }

```
运行结果：

```
jdk6
false
false

jdk7
false
true
```

这段代码在jdk6中输出是`false false`，但是在jdk7中输出的是`false true`。我们通过图来一行行解释。

**先来认识下intern()函数**：

　　intern函数的作用是将对应的符号常量进入特殊处理，在JDK1.6以前 和 JDK1.7以后有不同的处理；

　　在JDK1.6中，intern的处理是 先判断字符串常量是否在字符串常量池中，如果存在直接返回该常量，如果没有找到，则将该字符串常量加入到字符串常量区，也就是在字符串常量区建立该常量；

　　在JDK1.7中，intern的处理是 先判断字符串常量是否在字符串常量池中，如果存在直接返回该常量，如果没有找到，说明该字符串常量在堆中，则处理是把堆区该对象的引用加入到字符串常量池中，以后别人拿到的是该字符串常量的引用，实际存在堆中

**JDK1.6**

[![JDK1.6代码图](https://camo.githubusercontent.com/0b13ad545598ae4172eca142f335e8b96834d621758fc4b5383317bdb4747368/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31322f31362f313637623636336332363465366166323f696d61676556696577322f302f772f313238302f682f3936302f666f726d61742f776562702f69676e6f72652d6572726f722f31)](https://camo.githubusercontent.com/0b13ad545598ae4172eca142f335e8b96834d621758fc4b5383317bdb4747368/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31322f31362f313637623636336332363465366166323f696d61676556696577322f302f772f313238302f682f3936302f666f726d61742f776562702f69676e6f72652d6572726f722f31)

`String s = new String("2");`创建了两个对象，一个在堆中的StringObject对象，一个是在常量池中的“2”对象。 `s.intern();`在常量池中寻找与s变量内容相同的对象，发现已经存在内容相同对象“2”，返回对象2的地址。 `String s2 = "2";`使用字面量创建，在常量池寻找是否有相同内容的对象，发现有，返回对象"2"的地址。 `System.out.println(s == s2);`从上面可以分析出，s变量和s2变量地址指向的是不同的对象，所以返回false

`String s3 = new String("3") + new String("3");`创建了两个对象，一个在堆中的StringObject对象，一个是在常量池中的“3”对象。中间还有2个匿名的new String("3")我们不去讨论它们。 `s3.intern();`在常量池中寻找与s3变量内容相同的对象，没有发现“33”对象，在常量池中创建“33”对象，返回“33”对象的地址。 `String s4 = "33";`使用字面量创建，在常量池寻找是否有相同内容的对象，发现有，返回对象"33"的地址。 `System.out.println(s3 == s4);`从上面可以分析出，s3变量和s4变量地址指向的是不同的对象，所以返回false

**JDK1.7**

[![JDK1.7代码图](https://camo.githubusercontent.com/0eb51911a92684a0ce65681aca2de1036ca33ec0fd7dddb00583048ee1b6e671/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31322f31362f313637623636336332363137613139343f696d61676556696577322f302f772f313238302f682f3936302f666f726d61742f776562702f69676e6f72652d6572726f722f31)](https://camo.githubusercontent.com/0eb51911a92684a0ce65681aca2de1036ca33ec0fd7dddb00583048ee1b6e671/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31322f31362f313637623636336332363137613139343f696d61676556696577322f302f772f313238302f682f3936302f666f726d61742f776562702f69676e6f72652d6572726f722f31)

`String s = new String("2");`创建了两个对象，一个在堆中的StringObject对象，一个是在堆中的“2”对象，并在常量池中保存“2”对象的引用地址。 `s.intern();`在常量池中寻找与s变量内容相同的对象，发现已经存在内容相同对象“2”，返回对象“2”的引用地址。 `String s2 = "2";`使用字面量创建，在常量池寻找是否有相同内容的对象，发现有，返回对象“2”的引用地址。 `System.out.println(s == s2);`从上面可以分析出，s变量和s2变量地址指向的是不同的对象，所以返回false

`String s3 = new String("3") + new String("3");`创建了两个对象，一个在堆中的StringObject对象，一个是在堆中的“3”对象，并在常量池中保存“3”对象的引用地址。中间还有2个匿名的new String("3")我们不去讨论它们。 `s3.intern();`在常量池中寻找与s3变量内容相同的对象，没有发现“33”对象，将s3对应的StringObject对象的地址保存到常量池中，返回StringObject对象的地址。 `String s4 = "33";`使用字面量创建，在常量池寻找是否有相同内容的对象，发现有，返回其地址，也就是StringObject对象的引用地址。 `System.out.println(s3 == s4);`从上面可以分析出，s3变量和s4变量地址指向的是相同的对象，所以返回true。

## String 是最基本的数据类型吗?

不是。Java 中的基本数据类型只有 8 个 ：byte、short、int、long、float、double、char、boolean；除了基本类型（primitive type），剩下的都是引用类型（referencetype），Java 5 以后引入的枚举类型也算是一种比较特殊的引用类型。

## String有哪些特性?

-   不变性：String 是只读字符串，是一个典型的 immutable 对象，对它进行任何操作，其实都是创建一个新的对象，再把引用指向该对象。不变模式的主要作用在于当一个对象需要被多线程共享并频繁访问时，可以保证数据的一致性；
  
-   常量池优化：String 对象创建之后，会在字符串常量池中进行缓存，如果下次创建同样的对象时，会直接返回缓存的引用；
  
-   final：使用 final 来定义 String 类，表示 String 类不能被继承，提高了系统的安全性。
  
	
## String类能被继承吗，为什么？


​	
​	
## String,StringBuffer,StringBuilder区别

1、三者在执行速度上：StringBuilder > StringBuffer > String (由于String是常量，不可改变，拼接时会重新创建新的对象)。

2、StringBuffer是线程安全的，StringBuilder是线程不安全的。（由于**StringBuffer有缓冲**区）



## 如何将字符串反转?

使用 StringBuilder 或者 stringBuffer 的 reverse() 方法。

示例代码：

```java
// StringBuffer reverse
StringBuffer stringBuffer = new StringBuffer();
stringBuffer.append("abcdefg");
System.out.println(stringBuffer.reverse()); // gfedcba
// StringBuilder reverse
StringBuilder stringBuilder = new StringBuilder();
stringBuilder.append("abcdefg");
System.out.println(stringBuilder.reverse()); // gfedcba
```



## String 类的常用方法都有那些？

- indexOf()：返回指定字符的索引。
- charAt()：返回指定索引处的字符。
- replace()：字符串替换。
- trim()：去除字符串两端空白。
- split()：分割字符串，返回一个分割后的字符串数组。
- getBytes()：返回字符串的 byte 类型数组。
- length()：返回字符串长度。
- toLowerCase()：将字符串转成小写字母。
- toUpperCase()：将字符串转成大写字符。
- substring()：截取字符串。
- equals()：字符串比较。


## "abcde"字符占用多少内存？
https://cloud.tencent.com/developer/article/1734975

