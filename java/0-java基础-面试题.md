目标:: [[java基础-目标]]
分类:: 技术
目标状态:: 进行中
开始日期:: 2022.07.01
结束时间:: 2022.07.30
tags::`= this.file.tags`
#目标/技术

# 数据类型  [[Java中基础数据类型和引用类型的区别]]
## Java有哪些数据类型
Java**中**有 8 种基本数据类型，分别为：

-   **6 种数字类型 （四个整数形，两个浮点型）**：byte、short、int、long、float、double
  
-   **1 种字符类型**：char
  
-   **1 种布尔型**：boolean。
  

**byte：**

-   byte 数据类型是8位、有符号的，以二进制补码表示的整数；
-   最小值是 **-128（-2^7）**；
-   最大值是 **127（2^7-1）**；
-   默认值是 **0**；
-   byte 类型用在大型数组中节约空间，主要代替整数，因为 byte 变量占用的空间只有 int 类型的四分之一；
-   例子：byte a = 100，byte b = -50。

**short：**

-   short 数据类型是 16 位、有符号的以二进制补码表示的整数
-   最小值是 **-32768（-2^15）**；
-   最大值是 **32767（2^15 - 1）**；
-   Short 数据类型也可以像 byte 那样节省空间。一个short变量是int型变量所占空间的二分之一；
-   默认值是 **0**；
-   例子：short s = 1000，short r = -20000。

**int：**

-   int 数据类型是32位、有符号的以二进制补码表示的整数；
-   最小值是 **-2,147,483,648（-2^31）**；
-   最大值是 **2,147,483,647（2^31 - 1）**；
-   一般地整型变量默认为 int 类型；
-   默认值是 **0** ；
-   例子：int a = 100000, int b = -200000。

**long：**

-   **注意：Java 里使用 long 类型的数据一定要在数值后面加上 L，否则将作为整型解析**
  
-   long 数据类型是 64 位、有符号的以二进制补码表示的整数；
  
-   最小值是 **-9,223,372,036,854,775,808（-2^63）**；
  
-   最大值是 **9,223,372,036,854,775,807（2^63 -1）**；
  
-   这种类型主要使用在需要比较大整数的系统上；
  
-   默认值是 **0L**；
  
-   例子： long a = 100000L，Long b = -200000L。  
    "L"理论上不分大小写，但是若写成"l"容易与数字"1"混淆，不容易分辩。所以最好大写。
    

**float：**

-   float 数据类型是单精度、32位、符合IEEE 754标准的浮点数；
-   float 在储存大型浮点数组的时候可节省内存空间；
-   默认值是 **0.0f**；
-   浮点数不能用来表示精确的值，如货币；
-   例子：float f1 = 234.5f。

**double：**

-   double 数据类型是双精度、64 位、符合IEEE 754标准的浮点数；
-   浮点数的默认类型为double类型；
-   double类型同样不能表示精确的值，如货币；
-   默认值是 **0.0d**；
-   例子：double d1 = 123.4。

**char：**

-   char类型是一个单一的 16 位 Unicode 字符；
-   最小值是 **\u0000**（即为 0）；
-   最大值是 **\uffff**（即为 65535）；
-   char 数据类型可以储存任何字符；
-   例子：char letter = ‘A’;（**单引号**）

**boolean：**

-   boolean数据类型表示一位的信息；
-   只有两个取值：true 和 false；
-   这种类型只作为一种标志来记录 true/false 情况；
-   默认值是 **false**；
-   例子：boolean one = true。

**这八种基本类型都有对应的包装类分别为：Byte、Short、Integer、Long、Float、Double、Character、Boolean**

类型名称

字节、位数

最小值

最大值

默认值

例子

byte字节

1字节，8位

-128（-2^7）

127（2^7-1）

0

byte a = 100，byte b = -50

short短整型

2字节，16位

-32768（-2^15）

32767（2^15 - 1）

0

short s = 1000，short r = -20000

int整形

4字节，32位

-2,147,483,648（-2^31）

2,147,483,647（2^31 - 1）

0

int a = 100000, int b = -200000

lang长整型

8字节，64位

-9,223,372,036,854,775,808（-2^63）

9,223,372,036,854,775,807（2^63 -1）

0L

long a = 100000L，Long b = -200000L

double双精度

8字节，64位

double类型同样不能表示精确的值，如货币

0.0d

double d1 = 123.4

float单精度

4字节，32位

在储存大型浮点数组的时候可节省内存空间

不同统计精准的货币值

0.0f

float f1 = 234.5f

char字符

2字节，16位

\u0000（即为0）

\uffff（即为65,535）

可以储存任何字符

char letter = ‘A’;

boolean布尔

返回true和false两个值

这种类型只作为一种标志来记录 true/false 情况；

只有两个取值：true 和 false；

false

boolean one = true

## Java中引用数据类型有哪些，它们与基本数据类型有什么区别？

引用数据类型分3种：类，接口，数组；

**简单来说,只要不是基本数据类型.都是引用数据类型。 那他们有什么不同呢？**

### 1、从概念方面来说

1,基本数据类型:变量名指向具体的数值

2,引用数据类型:变量名不是指向具体的数值,而是指向存数据的内存地址,.也及时hash值

### 2、从内存的构建方面来说(内存中,有堆内存和栈内存两者)

1,基本数据类型:被创建时,在栈内存中会被划分出一定的内存,并将数值存储在该内存中.

2,引用数据类型:被创建时,首先会在栈内存中分配一块空间,然后在堆内存中也会分配一块具体的空间用来存储数据的具体信息,即hash值,然后由栈中引用指向堆中的对象地址.

**举个例子**

```java
//基本数据类型作为方法参数被调用
public class Main{
	public static void main(String[] args){
		//基本数据类型
		int i = 1;
		int j = 1;
		double d = 1.2;
 
		//引用数据类型
		String str = "Hello";
		String str1= "Hello";
	}
}
```

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210810095611.png)

由上图可知，基本数据类型中会存在两个相同的1，而引用型类型就不会存在相同的数据。  
假如"hello"的引用地址是xxxxx1，声明str变量并其赋值"hello"实际上就是让str变量引用了"hello"的内存地址，这个内存地址就存储在堆内存中，是不会改变的，当再次声明变量str1也是赋值为"hello"时，此时就会在堆内存中查询是否有"hello"这个地址，如果堆内存中已经存在这个地址了，就不会再次创建了，而是让str1变量也指向xxxxx1这个地址，如果没有的话，就会重新创建一个地址给str1变量。



## JAVA中的几种基本数据类型是什么，各自占用多少字节

## Java 引用类型的分类和区别
[[Java中基础数据类型和引用类型的区别]]


# 抽象类和接口
[[抽象类和接口]]


# final  [[3-final详解]]

## final关键字

https://duanqz.github.io/2015-07-02-Java-Practices-Use-Final-Liberally


## final 在 Java 中有什么作用？

final作为Java中的关键字可以用于三个地方。用于修饰类、类属性和类方法。

特征：凡是引用final关键字的地方皆不可修改！

(1)修饰类：表示该类不能被继承；

(2)修饰方法：表示方法不能被重写；

(3)修饰变量：表示变量只能一次赋值以后值不能被修改（常量）。

## final有哪些用法?

final也是很多面试喜欢问的地方,但我觉得这个问题很无聊,通常能回答下以下5点就不错了:

-   被final修饰的类不可以被继承
-   被final修饰的方法不可以被重写
-   被final修饰的变量不可以被改变.如果修饰引用,那么表示引用不可变,引用指向的内容可变.
-   被final修饰的方法,JVM会尝试将其内联,以提高运行效率
-   被final修饰的常量,在编译阶段会存入常量池中.

除此之外,编译器对final域要遵守的两个重排序规则更好:

在构造函数内对一个final域的写入,与随后把这个被构造对象的引用赋值给一个引用变量,这两个操作之间不能重排序 初次读一个包含final域的对象的引用,与随后初次读这个final域,这两个操作之间不能重排序.


## 关键字final和static是怎么使用的。

final:

1、final变量即为常量，只能赋值一次。

2、final方法不能被子类重写。

3、final类不能被继承。



static：

1、static变量：对于静态变量在内存中只有一个拷贝（节省内存），JVM只为静态分配一次内存，

在加载类的过程中完成静态变量的内存分配，可用类名直接访问（方便），当然也可以通过对象来访问（但是这是不推荐的）。



2、static代码块

 static代码块是类加载时，初始化自动执行的。



3、static方法

static方法可以直接通过类名调用，任何的实例也都可以调用，因此static方法中不能用this和super关键字，

不能直接访问所属类的实例变量和实例方法(就是不带static的成员变量和成员成员方法)，只能访问所属类的静态成员变量和成员方法。






# static

## static都有哪些用法?

所有的人都知道static关键字这两个基本的用法:静态变量和静态方法.也就是被static所修饰的变量/方法都属于类的静态资源,类实例所共享.

除了静态变量和静态方法之外,static也用于静态块,多用于初始化操作:

```java
public calss PreCache{
    static{
        //执行相关操作
    }
}
1.2.3.4.5.
```

此外static也多用于修饰内部类,此时称之为静态内部类.

最后一种用法就是静态导包,即`import static`.import static是在JDK 1.5之后引入的新特性,可以用来指定导入某个类中的静态资源,并且不需要使用类名,可以直接使用资源名,比如:

登录后复制

```java
import static java.lang.Math.*;
 
public class Test{
 
    public static void main(String[] args){
        //System.out.println(Math.sin(20));传统做法
        System.out.println(sin(20));
    }
}
1.2.3.4.5.6.7.8.9.
```

## static和final区别

| 关键词 | 修饰物 | 影响                                                     |
| :----- | :----- | :------------------------------------------------------- |
| final  | 变量   | 分配到常量池中，程序不可改变其值                         |
| final  | 方法   | 子类中将不能被重写                                       |
| final  | 类     | 不能被继承                                               |
| static | 变量   | 分配在内存堆上，引用都会指向这一个地址而不会重新分配内存 |
| static | 方法块 | 虚拟机优先加载                                           |
| static | 类     | 可以直接通过类来调用而不需要new                          |


# 面向对象三大特性  [[面向对象三大特征-封装-继承-多态]]
封装、继承、多态


## 继承与多态



## Java中重载和重写的区别：

1、重载：一个类中可以有多个相同方法名的，但是参数类型和个数都不一样。这是重载。

2、重写：子类继承父类，则子类可以通过实现父类中的方法，从而新的方法把父类旧的方法覆盖。





# hashcode  [[java中==-equals-hashCode]]
## 使用方面：==、equals

* 基本数据类型:判断数据是否相等，用==和!=判断。
* 引用数据类型:判断数据是否相等，用equals()方法,==和!=是比较数值的。而equals()方法是比较内存地址的。

**补充：数据类型选择的原则**

- 如果要表示整数就使用int，表示小数就使用double；
- 如果要描述日期时间数字或者表示文件（或内存）大小用long；
- 如果要实现内容传递或者编码转换使用byte；
- 如果要实现逻辑的控制，可以使用booleam；
- 如果要使用中文，使用char避免中文乱码；
- 如果按照保存范围：byte < int < long < double;



##  String 和 Integer 等重写了 equals 方法，把它变成了值比较

首先来看默认情况下 equals 比较一个有相同值的对象，代码如下：

```java
class Cat {
    public Cat(String name) {
        this.name = name;
    }

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

Cat c1 = new Cat("叶痕秋");
Cat c2 = new Cat("叶痕秋");
System.out.println(c1.equals(c2)); // false
```

输出结果出乎我们的意料，竟然是 false？这是怎么回事，看了 equals 源码就知道了，源码如下：

```java
public boolean equals(Object obj) {
        return (this == obj);
}
```

原来 equals 本质上就是 ==。

那问题来了，两个相同值的 String 对象，为什么返回的是 true？代码如下：

```java
String s1 = new String("叶子");
String s2 = new String("叶子");
System.out.println(s1.equals(s2)); // true
```

同样的，当我们进入 String 的 equals 方法，找到了答案，代码如下：

```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

原来是 String 重写了 Object 的 equals 方法，把引用比较改成了值比较。



**总结**

**== 对于基本类型来说是值比较，对于引用类型来说是比较的是引用；而 equals 默认情况下是引用比较，只是很多类重新了 equals 方法，比如 String、Integer 等把它变成了值比较，所以一般情况下 equals 比较的是值是否相等。**




## hashCode 和对象的内存地址有什么关系？[[hashCode和对象的内存地址有什么关系]]
会输出该类的全限定类名和一串字符串：

```
java.lang.Object@6659c656
```

其实`@`后面的**只是对象的 hashcode 值，16进制展示的 hashcode 而已**


## Hashcode的作用

java的集合有两类，一类是List，还有一类是Set。前者有序可重复，后者无序不重复。当我们在set中插入的时候怎么判断是否已经存在该元素呢，可以通过**equals方法**。但是如果**元素太多**，用这样的方法就会比较满。

于是有人发明了哈希算法来提高集合中查找元素的效率。 这种方式将集合分成若干个存储区域，每个对象可以计算出一个哈希码，可以将哈希码分组，每组分别对应某个存储区域，根据一个对象的哈希码就可以确定该对象应该存储的那个区域。

hashCode方法可以这样理解：它返回的就是根据对象的内存地址换算出的一个值。这样一来，当集合要添加新的元素时，先调用这个元素的hashCode方法，就一下子能定位到它应该放置的物理位置上。如果这个位置上没有元素，它就可以直接存储在这个位置上，不用再进行任何比较了；**如果这个位置上已经有元素了，就调用它的equals方法与新元素进行比较，相同的话就不存了，不相同就散列其它的地址。这样一来实际调用equals方法的次数就大大降低了，几乎只需要一两次**。

## hashcode 与 equal 区别?

（1）**关系操作符 ==**

- 若操作数的类型是基本数据类型，则该关系操作符判断的是左右两边操作数的值是否相等
- 若操作数的类型是引用数据类型，则该关系操作符判断的是左右两边操作数的**内存地址**是否相同。**也就是说，若此时返回 \**true\**, 则该操作符作用的一定是同一个对象**

------

（2）**equals（内部实现三个步骤）**

- 先 比较引用是否相同 (是否为同一对象)
- 再 **判断类型**是否一致（是否为同一类型）
- 最后 比较内容是否一致
- 注：equal 的默认行为是比较**引用**，所以除非在自己的新类中覆盖了 equal() 方法，否则不可能表现出我们希望的行为

------

（3）**hashCode**

- hashcode 是系统用来快速检索对象而使用（一般在需要用哈希算法的数据结构中才有用，比如 HashSet, HashMap 和 Hashtable）
- 重写 equals 方法和 hashcode 方法时，**equals方法中用到的成员变量也必定会在 hashcode方法中用到**，只不过前者作为比较项，后者作为生成摘要的信息项，本质上所用到的数据是一样的，从而保证二者的一致性

------

（4）equals 与 hashCode 关系

- 如果两个对象 equals，那么它们的 hashCode 必然相等
- 但是 hashCode 相等，equals 不一定相等

------

传送门： [Java 中的 ==, equals 与 hashCode 的区别与联系](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fjustloveyou_%2Farticle%2Fdetails%2F52464440)

## 两个值相等的 Integer 对象，== 比较，判断是否相等？


## 两个对象的 hashCode() 相同， 那么 equals() 也一定为 true吗？

不对，两个对象的 hashCode() 相同，equals() 不一定 true。

代码示例：

```java
String str1 = "keep";
String str2 = "brother";
System. out. println(String. format("str1：%d | str2：%d",  str1. hashCode(),str2. hashCode()));
System. out. println(str1. equals(str2));
1.2.3.4.
```

执行结果:

```html
str1：1179395 | str2：1179395

false
1.2.3.
```

代码解读：很显然“keep”和“brother”的 hashCode() 相同，然而 equals() 则为 false，因为在散列表中，**hashCode() 相等即两个键值对的哈希值相等，然而哈希值相等，并不一定能得出键值对相等**。

## 如何在父类中为子类自动完成所有的hashcode和equals实现？这么做有何优劣

## 这样的a.hashcode() 有什么用，与a.equals(b)有什么关系

## 有没有可能2个不相等的对象有相同的hashcode



#  能将 int 强制转换为 byte 类型的变量吗？如果该值大于 byte 类型的范围，将会出现什么现象？
我们可以做强制转换，但是 Java 中 int 是 32 位的，而 byte 是 8 位的，所以，如果强制转化，int 类型的高 24 位将会被丢弃，因为byte 类型的范围是从 -128 到 127



# 包装类型   [[自动装箱和拆箱]]

## 包装类型是什么？基本类型和包装类型有什么区别？

Java 为每一个基本数据类型都引入了对应的包装类型（wrapper class），int 的包装类就是 Integer，从 Java 5 开始引入了自动装箱/拆箱机制，把基本类型转换成包装类型的过程叫做装箱（boxing）；反之，把包装类型转换成基本类型的过程叫做拆箱（unboxing），使得二者可以相互转换。

Java 为每个原始类型提供了包装类型：

原始类型: boolean，char，byte，short，int，long，float，double

包装类型：Boolean，Character，Byte，Short，Integer，Long，Float，Double

**基本类型和包装类型的区别主要有以下 几点**：

-   **包装类型可以为 null，而基本类型不可以**。它使得包装类型可以应用于 POJO 中，而基本类型则不行。那为什么 POJO 的属性必须要用包装类型呢？《阿里巴巴 Java 开发手册》上有详细的说明， 数据库的查询结果可能是 null，如果使用基本类型的话，因为要自动拆箱（将包装类型转为基本类型，比如说把 Integer 对象转换成 int 值），就会抛出 `NullPointerException` 的异常。
  
-   **包装类型可用于泛型，而基本类型不可以**。泛型不能使用基本类型，因为使用基本类型时会编译出错。
  
    List\<int\\> list = new ArrayList\<\\>(); // 提示 Syntax error, insert "Dimensions" to complete ReferenceType
    List\<Integer\\> list = new ArrayList\<\\>();
    
    因为泛型在编译时会进行类型擦除，最后只保留原始类型，而原始类型只能是 Object 类及其子类——基本类型是个特例。
    
-   **基本类型比包装类型更高效**。基本类型在栈中直接存储的具体数值，而包装类型则存储的是堆中的引用。 很显然，相比较于基本类型而言，包装类型需要占用更多的内存空间。
  

## 解释一下自动装箱和自动拆箱？

**自动装箱：将基本数据类型重新转化为对象**

```java
    public class Test {  
        public static void main(String[] args) {  
            // 声明一个Integer对象，用到了自动的装箱：解析为:Integer num = Integer.valueOf(9);
            Integer num = 9;
        }  
    } 
``` 

9是属于基本数据类型的，原则上它是不能直接赋值给一个对象Integer的。但jdk1.5 开始引入了自动装箱/拆箱机制，就可以进行这样的声明，自动将基本数据类型转化为对应的封装类型，成为一个对象以后就可以调用对象所声明的所有的方法。

**自动拆箱：将对象重新转化为基本数据类型**

```java
 public class Test {  
        public static void main(String[] args) {  
            / /声明一个Integer对象
	        Integer num = 9;
            
            // 进行计算时隐含的有自动拆箱
    	    System.out.print(num--);
        }  
    }
```  

因为**对象时不能直接进行运算的，而是要转化为基本数据类型后才能进行加减乘除**。

## int 和 Integer 有什么区别?

-   Integer是int的包装类；int是基本数据类型；
-   Integer变量必须实例化后才能使用；int变量不需要；
-   Integer实际是对象的引用，指向此new的Integer对象；int是直接存储数据值 ；
-   Integer的默认值是null；int的默认值是0。

## 两个new生成的Integer变量的对比

由于Integer变量实际上是对一个Integer对象的引用，所以两个通过new生成的Integer变量永远是不相等的（因为new生成的是两个对象，其内存地址不同）。

```java
Integer i = new Integer(10000);
Integer j = new Integer(10000);
System.out.print(i == j); //false
```

## Integer变量和int变量的对比

Integer变量和int变量比较时，只要两个变量的值是向等的，则结果为true（因为包装类Integer和基本数据类型int比较时，java会自动拆包装为int，然后进行比较，实际上就变为两个int变量的比较）

```java
    int a = 10000;
    Integer b = new Integer(10000);
    Integer c=10000;
    System.out.println(a == b); // true
    System.out.println(a == c); // true
```

## 非new生成的Integer变量和new Integer()生成变量的对比

非new生成的Integer变量和new Integer()生成的变量比较时，结果为false。（因为非new生成的Integer变量指向的是java常量池中的对象，而new Integer()生成的变量指向堆中新建的对象，两者在内存中的地址不同）

```java
    Integer b = new Integer(10000);
    Integer c=10000;
    System.out.println(b == c); // false
```

## 两个非new生成的Integer对象的对比

对于两个非new生成的Integer对象，进行比较时，如果两个变量的值在区间-128到127之间，则比较结果为true，如果两个变量的值不在此区间，则比较结果为false

```java
Integer i = 100;
Integer j = 100;
System.out.print(i == j); //true

Integer i = 128;
Integer j = 128;
System.out.print(i == j); //false
```

当值在 -128 ~ 127之间时，java会进行自动装箱，然后会对值进行缓存，如果下次再有相同的值，会直接在缓存中取出使用。缓存是通过Integer的内部类IntegerCache来完成的。当值超出此范围，会在堆中new出一个对象来存储。

给一个Integer对象赋一个int值的时候，会调用Integer类的静态方法valueOf，源码如下：

```java
public static Integer valueOf(String s, int radix) throws NumberFormatException {
        return Integer.valueOf(parseInt(s,radix));
    }
```

/**
 * （1）在-128~127之内：静态常量池中cache数组是static final类型，cache数组对象会被存储于静态常量池中。
 * cache数组里面的元素却不是static final类型，而是cache[k] = new Integer(j++)，
 * 那么这些元素是存储于堆中，只是cache数组对象存储的是指向了堆中的Integer对象（引用地址）
 * 
 * （2）在-128~127 之外：新建一个 Integer对象，并返回。
```java
 */
    public static Integer valueOf(int i) {
        assert IntegerCache.high \>= 127;
        if (i \>= IntegerCache.low && i <= IntegerCache.high) {
            return IntegerCache.cache[i + (-IntegerCache.low)];
        }
        return new Integer(i);
    }
```

IntegerCache是Integer的内部类，源码如下：

     /**
 * 缓存支持自动装箱的对象标识语义 -128和127（含）。
 * 缓存在第一次使用时初始化。 缓存的大小可以由-XX：AutoBoxCacheMax = \<size\\>选项控制。
 * 在VM初始化期间，java.lang.Integer.IntegerCache.high属性可以设置并保存在私有系统属性中
```java
 */
    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            }
            high = h;
        
            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++) {
                cache[k] = new Integer(j++); // 创建一个对象
            }
        }
        
        private IntegerCache() {}
    }
```

# 泛型   [[1-Java泛型]]

## 泛型常用特点

泛型是Java SE 1.5之后的特性， 《Java 核心技术》中对泛型的定义是：

\> “泛型” 意味着编写的代码可以被不同类型的对象所重用。

“泛型”，顾名思义，“泛指的类型”。我们提供了泛指的概念，但具体执行的时候却可以有具体的规则来约束，比如我们用的非常多的ArrayList就是个泛型类，ArrayList作为集合可以存放各种元素，如Integer, String，自定义的各种类型等，但在我们使用的时候通过具体的规则来约束，如我们可以约束集合中只存放Integer类型的元素，如

```java
List<Integer> iniData = new ArrayList<>()
```

**使用泛型的好处？**

以集合来举例，使用泛型的好处是我们不必因为添加元素类型的不同而定义不同类型的集合，如整型集合类，浮点型集合类，字符串集合类，我们可以定义一个集合来存放整型、浮点型，字符串型数据，而这并不是最重要的，因为我们只要把底层存储设置了Object即可，添加的数据全部都可向上转型为Object。 更重要的是我们可以通过规则按照自己的想法控制存储的数据类型。



## Java中的泛型是什么 ?

泛型是 JDK1.5 的一个新特性，**泛型就是将类型参数化，其在编译时才确定具体的参数。**这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法。

## 使用泛型的好处是什么?

远在 JDK 1.4 版本的时候，那时候是没有泛型的概念的，如果使用 Object 来实现通用、不同类型的处理，有这么两个缺点：

1.  每次使用时都需要强制转换成想要的类型
2.  在编译时编译器并不知道类型转换是否正常，运行时才知道，不安全。

如这个例子：

List list = new ArrayList();
list.add("www.cnblogs.com");
list.add(23);
String name = (String)list.get(0);
String number = (String)list.get(1);	//ClassCastException

上面的代码在运行时会发生强制类型转换异常。这是因为我们在存入的时候，第二个是一个 Integer 类型，但是取出来的时候却将其强制转换为 String 类型了。Sun 公司为了使 Java 语言更加安全，减少运行时异常的发生。于是在 JDK 1.5 之后推出了泛型的概念。

根据《Java 编程思想》中的描述，泛型出现的动机在于：有许多原因促成了泛型的出现，而最引人注意的一个原因，就是**为了创建容器类**。

**使用泛型的好处有以下几点**：

1.  类型安全
  
    -   泛型的主要目标是提高 Java 程序的类型安全
    -   编译时期就可以检查出因 Java 类型不正确导致的 ClassCastException 异常
    -   符合越早出错代价越小原则
2.  消除强制类型转换
  
    -   泛型的一个附带好处是，使用时直接得到目标类型，消除许多强制类型转换
    -   所得即所需，这使得代码更加可读，并且减少了出错机会
3.  潜在的性能收益
  
    -   由于泛型的实现方式，支持泛型（几乎）不需要 JVM 或类文件更改
    -   所有工作都在编译器中完成
    -   编译器生成的代码跟不使用泛型（和强制类型转换）时所写的代码几乎一致，只是更能确保类型安全而已

## Java泛型的原理是什么 ? 什么是类型擦除 ?

泛型是一种语法糖，泛型这种语法糖的基本原理是类型擦除。Java中的泛型基本上都是在编译器这个层次来实现的，也就是说：**泛型只存在于编译阶段，而不存在于运行阶段。**在编译后的 class 文件中，是没有泛型这个概念的。

类型擦除：使用泛型的时候加上的类型参数，编译器在编译的时候去掉类型参数。

例如：

```
public class Caculate<T\> {
    private T num;
}
```

　　我们定义了一个泛型类，定义了一个属性成员，该成员的类型是一个泛型类型，这个 T 具体是什么类型，我们也不知道，它只是用于限定类型的。反编译一下这个 Caculate 类：

```
public class Caculate{
    public Caculate(){}
    private Object num;
}
```

　　发现编译器擦除 Caculate 类后面的两个尖括号，并且将 num 的类型定义为 Object 类型。

　　那么是不是所有的泛型类型都以 Object 进行擦除呢？大部分情况下，泛型类型都会以 Object 进行替换，而有一种情况则不是。那就是使用到了extends和super语法的有界类型，如：

```
public class Caculate<T extends String> {
    private T num;
}
```

　　这种情况的泛型类型，num 会被替换为 String 而不再是 Object。这是一个类型限定的语法，它限定 T 是 String 或者 String 的子类，也就是你构建 Caculate 实例的时候只能限定 T 为 String 或者 String 的子类，所以无论你限定 T 为什么类型，String 都是父类，不会出现类型不匹配的问题，于是可以使用 String 进行类型擦除。

　　实际上编译器会正常的将使用泛型的地方编译并进行类型擦除，然后返回实例。但是除此之外的是，如果构建泛型实例时使用了泛型语法，那么编译器将标记该实例并关注该实例后续所有方法的调用，每次调用前都进行安全检查，非指定类型的方法都不能调用成功。

　　实际上编译器不仅关注一个泛型方法的调用，它还会为某些返回值为限定的泛型类型的方法进行强制类型转换，由于类型擦除，返回值为泛型类型的方法都会擦除成 Object 类型，当这些方法被调用后，编译器会额外插入一行 checkcast 指令用于强制类型转换。这一个过程就叫做『泛型翻译』。

## 什么是泛型中的限定通配符和非限定通配符 ?

限定通配符对类型进行了限制。有两种限定通配符，一种是\<? extends T\>它通过确保类型必须是T的子类来设定类型的上界，另一种是\<? super T\>它通过确保类型必须是T的父类来设定类型的下界。泛型类型必须用限定内的类型来进行初始化，否则会导致编译错误。

非限定通配符 **？**,可以用任意类型来替代。如`List<?>` 的意思是这个集合是一个可以持有任意类型的集合，它可以是`List<A>`，也可以是`List<B>`,或者`List<C>`等等。

## List\<? extends T\>和List \<? super T\>之间有什么区别 ?

这两个List的声明都是限定通配符的例子，List\<? extends T\>可以接受任何继承自T的类型的List，而List\<? super T\>可以接受任何T的父类构成的List。例如List\<? extends Number\>可以接受List或List。

## 可以把List`<String>`传递给一个接受List`<Object>`参数的方法吗？

不可以。真这样做的话会导致编译错误。因为List可以存储任何类型的对象包括String, Integer等等，而List却只能用来存储String。　

List\<Object\> objectList;
List\<String\> stringList;
objectList = stringList;  //compilation error incompatible types


## Array中可以用泛型吗?

不可以。这也是为什么 Joshua Bloch 在 《Effective Java》一书中建议使用 List 来代替 Array，因为 List 可以提供编译期的类型安全保证，而 Array 却不能。

## 判断`ArrayList<String>`与`ArrayList<Integer>`是否相等？

ArrayList\<String\> a = new ArrayList\<String\>();
ArrayList\<Integer\> b = new ArrayList\<Integer\>();
Class c1 = a.getClass();
Class c2 = b.getClass();
System.out.println(c1 == c2); 

输出的结果是 true。因为无论对于 ArrayList 还是 ArrayList，它们的 Class 类型都是一直的，都是 ArrayList.class。

那它们声明时指定的 String 和 Integer 到底体现在哪里呢？

**答案是体现在类编译的时候。** 当 JVM 进行类编译时，会进行泛型检查，如果一个集合被声明为 String 类型，那么它往该集合存取数据的时候就会对数据进行判断，从而避免存入或取出错误的数据。


## 泛型擦除发生的时机是什么时候？

\> 在**编译期。**[我眼中的Java-Type体系](https://www.jianshu.com/p/e8eeff12c306)
\> 
\> [https://www.jianshu.com/p/e8eeff12c306](https://www.jianshu.com/p/e8eeff12c306)


## ArrayList和LinkendList区别，List泛型擦除，为什么反射能够在ArrayList< String \>中添加int类型


# IO

## Java的IO 流分为几种？

-   按照流的方向：输入流（inputStream）和输出流（outputStream）；
-   按照实现功能分：节点流（可以从或向一个特定的地方读写数据，如 FileReader）和处理流（是对一个已存在的流的连接和封装，通过所封装的流的功能调用实现数据读写， BufferedReader）；
-   按照处理数据的单位： 字节流和字符流。分别由四个抽象类来表示（每种流包括输入和输出两种所以一共四个）:InputStream，OutputStream，Reader，Writer。Java中其他多种多样变化的流均是由它们派生出来的。

[![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210810114318.png)](https://camo.githubusercontent.com/c41f1b5817353b6cec92fa34c4d97cdb8c294d0e965d9103ef700d900511b756/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f696d6167652d32303231303232373131333330313539332e706e67)

## 字节流如何转为字符流？

字节输入流转字符输入流通过 InputStreamReader 实现，该类的构造函数可以传入 InputStream 对象。

字节输出流转字符输出流通过 OutputStreamWriter 实现，该类的构造函数可以传入 OutputStream 对象。

## 字符流与字节流的区别？

-   读写的时候字节流是按字节读写，字符流按字符读写。
-   字节流适合所有类型文件的数据传输，因为计算机字节（Byte）是电脑中表示信息含义的最小单位。字符流只能够处理纯文本数据，其他类型数据不行，但是字符流处理文本要比字节流处理文本要方便。
-   在读写文件需要对内容按行处理，比如比较特定字符，处理某一行数据的时候一般会选择字符流。
-   只是读写文件，和文件内容无关时，一般选择字节流。

## BIO、NIO、AIO的区别？

-   BIO：同步并阻塞，在服务器中实现的模式为**一个连接一个线程**。也就是说，客户端有连接请求的时候，服务器就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，当然这也可以通过线程池机制改善。BIO**一般适用于连接数目小且固定的架构**，这种方式对于服务器资源要求比较高，而且并发局限于应用中，是JDK1.4之前的唯一选择，但好在程序直观简单，易理解。
-   NIO：同步并非阻塞，在服务器中实现的模式为**一个请求一个线程**，也就是说，客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到有连接IO请求时才会启动一个线程进行处理。**NIO一般适用于连接数目多且连接比较短（轻操作）的架构**，并发局限于应用中，编程比较复杂，从JDK1.4开始支持。
-   AIO：异步并非阻塞，在服务器中实现的模式为**一个有效请求一个线程**，也就是说，客户端的IO请求都是通过操作系统先完成之后，再通知服务器应用去启动线程进行处理。AIO一般适用于连接数目多且连接比较长（重操作）的架构，充分调用操作系统参与并发操作，编程比较复杂，从JDK1.7开始支持。

## Java IO都有哪些设计模式？

使用了**适配器模式**和**装饰器模式**

**适配器模式**：

Reader reader = new INputStreamReader(inputStream);

**把一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口不匹配而无法在一起工作的两个类能够在一起工作**

-   **类适配器**：Adapter类（适配器）继承Adaptee类（源角色）实现Target接口（目标角色）
-   **对象适配器**：Adapter类（适配器）持有Adaptee类（源角色）对象实例，实现Target接口（目标角色） [![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210810114308.png)](https://camo.githubusercontent.com/ab11d89e70be58f7b64be1d9a0d95c22cc9eb67a57ce9d2dae132c338b4f305d/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f696d6167652d32303231303232373131343931393330372e706e67)

**装饰器模式**：

new BufferedInputStream(new FileInputStream(inputStream));

**一种动态地往一个类中添加新的行为的设计模式。就功能而言，装饰器模式相比生成子类更为灵活，这样可以给某个对象而不是整个类添加一些功能。**

-   ConcreteComponent（具体对象）和Decorator（抽象装饰器）实现相同的Conponent（接口）并且Decorator（抽象装饰器）里面持有Conponent（接口）对象，可以传递请求。
-   ConcreteComponent（具体装饰器）覆盖Decorator（抽象装饰器）的方法并用super进行调用，传递请求。

[![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210810114258.png)](https://camo.githubusercontent.com/319c4196311786ada5e47afe053369e1dbf1399b6e93eeb80d4e101264282178/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f696d6167652d32303231303232373131353034303939392e706e67)





## java 中 IO 流分为几种？



按功能来分：输入流（input）、输出流（output）。



按类型来分：字节流和字符流。



字节流和字符流的区别是：字节流按 8 位传输以字节为单位输入输出数据，字符流按 16 位传输以字符为单位输入输出数据。



## BIO、NIO、AIO 有什么区别？



- BIO：Block IO 同步阻塞式 IO，就是我们平常使用的传统 IO，它的特点是模式简单使用方便，并发处理能力低。
- NIO：New IO 同步非阻塞 IO，是传统 IO 的升级，客户端和服务器端通过 Channel（通道）通讯，实现了多路复用。
- AIO：Asynchronous IO 是 NIO 的升级，也叫 NIO2，实现了异步非堵塞 IO ，异步 IO 的操作基于事件和回调机制。

## Files的常用方法都有哪些？



- Files.exists()：检测文件路径是否存在。
- Files.createFile()：创建文件。
- Files.createDirectory()：创建文件夹。
- Files.delete()：删除一个文件或目录。
- Files.copy()：复制文件。
- Files.move()：移动文件。
- Files.size()：查看文件个数。
- Files.read()：读取文件。
- Files.write()：写入文件。

# 集合框架

\> 1. HashMap。几乎每家公司都问，主要是内部原理如hash算法、冲突解决方案、扩容方案、红黑树的优缺点等。必会的内容，不会就直接当场去世了。
\> 2. HashSet。内部使用HashMap来实现，value设置为object。记住这个就好了。
\> 3. ConcurrentHashMap。必问。他的并发原理以及好处，同时有些面试官也会问缺点等问题。
\> 4. Hashtable、SychronizeMap。一般和ConcurrentHashMap一起问，进行对比。
\> 5. CopyOnWriteArrayList。一般会作为线程安全方法来进行比较优缺点。
\> 6. 集合框架重点还是在Map，但是其他的框架List和queue的原理也是要了解的。

- 访问限制符

  \> public protect default private 四个要懂，基础知识了。（笔者就是忽略了这些当时回答错了）特别注意protect是可以跨包访问的。

# 类

  \> 1. 4种内部类，特别注意每个class编译后都会产生一个class文件，不管静态或非静态。面试踩坑了
  \> 2. lambda的本质。就是匿名内部类。
  \> 3. 抽象类和接口的区别。这个很看理解，如果有开发过具体项目的会回答得更加深刻，这是背八股文体现不出来的。

# 异常

  \> 1. 异常体系、分类、机制。
  \> 2. 与error的区别。

- IO

  \> 主要还是问NIO的原理以及优缺点。建议把缓冲流的原理也得学一学并进行比较。

# 线程池
例如线程池，可以从参数作用、到线程池原理、到阻塞唤醒机制、到实际项目的参数配置，有非常多的知识点可以考察。因而这一块就看各位的造诣了。

  \> 1. 内部原理。必会的啊。
  \> 2. 关键参数作用及如何配置。重点在如何配置，需要结合具体的机器情况、任务情况等等考量。
  \> 3. 线程池的作用。不仅仅只是线程复用，更重要的是管理线程、控制线程数量。这个也比较考察具体的项目运用理解。
  \> 4. 常见的四种线程池。

# 并发

  \> 1. sychronize。必问，java的锁机制。特别是jdk6之后的锁优化以及运用场景。为什么是重量级的、JVM层如何实现如果了解可以加分。
  \> 2. Lock。必问，AQS的原理最好懂。一般会拿来和synchronize比较。
  \> 3. volatile。必问，会拿来和锁比较，他的两个重要作用。更深点会问到cpu缓存一致性协议、以及指令重排的类型与原理。
  \> 4. CAS。必问，问原理以及ABA问题。
  \> 5. 死锁。一般询问如何解决或者产生的条件。
  \> 6. Object的wait和notify。阻塞唤醒，一般会用一个代码或者具体的场景来询问如何保证多线程同步。
  \> 7. ThreadLocal。原理、内存泄露等
  \> 8. 这一块问的还是比较多，而且大都可以深入去问，看自己的学习程度了。

# JVM

  \> 1. GC机制。必问。
  \> 2. 类加载机制。必问，同时还会问双亲委托机制。
  \> 3. 方法调用过程。这个也问的挺多，也看对JVM的学习程度了。
  \> 4. 线程与进程的内存关系。如一个线程占多少内存、一个进程可以开多少线程、一个进程占用多少内存等。
  \> 5. 内存分布。JMM、运行时数据区、native内存分布。很看对JVM的理解程度。




# 参考

https://github.com/cosen1024/Java-Interview/blob/main/Java%E5%9F%BA%E7%A1%80/Java%E5%9F%BA%E7%A1%80%E4%B8%8B.md



[https://juejin.cn/post/6844903741032759310](https://juejin.cn/post/6844903741032759310)

[https://blog.csdn.net/chenliguan/article/details/53888018](https://blog.csdn.net/chenliguan/article/details/53888018)

[https://segmentfault.com/a/1190000010162647](https://segmentfault.com/a/1190000010162647)

[https://juejin.cn/post/6856664924203663367](https://juejin.cn/post/6856664924203663367)

[http://www.fanyilun.me/2015/10/29/Java%E5%8F%8D%E5%B0%84%E5%8E%9F%E7%90%86/](http://www.fanyilun.me/2015/10/29/Java%E5%8F%8D%E5%B0%84%E5%8E%9F%E7%90%86/)

[https://www.cnblogs.com/lixuwu/p/10829368.html](https://www.cnblogs.com/lixuwu/p/10829368.html)

[https://juejin.cn/post/6844903848167866375](https://juejin.cn/post/6844903848167866375)

[https://blog.csdn.net/lanzhupi/article/details/109809836](https://blog.csdn.net/lanzhupi/article/details/109809836)

[https://juejin.cn/post/6844903746682486791](https://juejin.cn/post/6844903746682486791)

[https://blog.csdn.net/ThinkWon/article/details/101681073](https://blog.csdn.net/ThinkWon/article/details/101681073)

[https://simplesnippets.tech/exception-handling-in-java-part-1/](https://simplesnippets.tech/exception-handling-in-java-part-1/)

[https://www.cnblogs.com/xingyunblog/p/8688859.html](https://www.cnblogs.com/xingyunblog/p/8688859.html)

[https://mp.weixin.qq.com/s/p5qM2UJ1uIWyongfVpRbCg](https://mp.weixin.qq.com/s/p5qM2UJ1uIWyongfVpRbCg)

[https://juejin.cn/post/6844903520856965128](https://juejin.cn/post/6844903520856965128)