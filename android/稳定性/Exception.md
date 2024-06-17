---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 线索

![image-20210511171229164](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210511171229.png)


异常是什么、异常组成、crash流程、怎么避免异常？

Exception是什么？为什么要有Exception？(Exception的作用)

怎么定义一个Exception？

什么时候出现异常？出现了异常后怎么办？

throw ….

怎么捕获一个Exception？

Try….catch….finally

Thread.setUncau

Thread.getUn

Exception应用？
# 2 异常

## 2.1 为什么要使用异常？(解决了什么问题)

> 首先我们可以明确一点就是异常的处理机制可以确保我们程序的健壮性，提高系统可用率。虽然我们不是特别喜欢看到它，但是我们不能不承认它的地位，作用。

在**没有异常机制**的时候我们是这样处理的：通过函数的返回值来判断是否发生了异常（这个返回值通常是已经约定好了的），调用**该函数的程序负责检查并且分析返回值**。虽然可以解决异常问题，但是这样做存在几个缺陷：

> 1、 容易混淆。如果约定返回值为-11111时表示出现异常，那么当程序最后的计算结果真的为-1111呢？
> 2、 代码可读性差。将异常处理代码和程序代码混淆在一起将会降低代码的可读性。
> 3、 由调用函数来分析异常，这要求程序员对库函数有很深的了解。

在OO中提供的异常处理机制是**提供代码健壮**的强有力的方式。使用异常机制它能够降低错误处理代码的复杂度，如果不使用异常，那么就必须检查特定的错误，并在程序中的许多地方去处理它。

而如果使用异常，那就不必在方法调用处进行检查，因为异常机制将保证能够捕获这个错误，并且，只需在一个地方处理错误，即所谓的异常处理程序中。

这种方式不仅节约代码，而且**把“概述在正常执行过程中做什么事”的代码和“出了问题怎么办”的代码相分离**。总之，与以前的错误处理方法相比，异常机制使代码的阅读、编写和调试工作更加井井有条。（摘自《Think in java 》）


## 2.2 是什么？（异常基本定义）

Java的异常处理机制很大一部分来自C++。它允许程序员**跳过暂时无法处理的问题**，以继续后续的开发，或者让程序**根据异常做出更加聪明的处理**。

Java使用一些特殊的对象来代表异常状况，这样对象称为异常对象。当异常状况发生时，Java会根据预先的设定，抛出(throw)代表当前状况的对象。所谓的**抛出是一种特殊的返回方式**。**该线程会暂停，逐层退出方法调用，直到遇到异常处理器(Exception Handler)。** 异常处理器可以捕捉(catch)的异常对象，并根据对象来决定下一步的行动，比如:

- 提醒用户
- 处理异常
- 继续程序
- 退出程序

> 在《Think in java》中是这样定义异常的：异常情形是指阻止当前方法或者作用域继续执行的问题。在这里一定要明确一点：异常代码某种程度的错误，尽管Java有异常处理机制，但是我们不能以“正常”的眼光来看待异常，异常处理机制的原因就是告诉你：这里可能会或者已经产生了错误，您的程序出现了不正常的情况，可能会导致程序失败！
>
> 那么**什么时候才会出现异常**呢？只有在你当前的环境下程序无法正常运行下去，也就是说程序已经无法来正确解决问题了，这时它所就会从当前环境中跳出，并抛出异常。**抛出异常后，它首先会做几件事**。
>
> 首先，它会使用new创建一个异常对象，然后在产生异常的位置终止程序，并且从当前环境中弹出对异常对象的引用，这时。异常处理机制就会接管程序，并开始寻找一个恰当的地方来继续执行程序，这个恰当的地方就是异常处理程序。
>
> 总的来说异常处理机制就是当程序发生异常时，它强制终止程序运行，记录异常信息并将这些信息反馈给我们，由我们来确定是否处理异常。


### 2.2.1 **异常的类型-异常的层次结构**

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427162223.png)



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210611063000.png)


橙色: unchecked; 蓝色: checked





![1354439580_6933](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210611070811.png)



Java把异常作为一种类，当做对象来处理。所有异常类的基类是Throwable类，两大子类分别是Error和Exception。

　　系统错误由Java虚拟机抛出，用**Error类表示。Error类描述的是内部系统错误**，例如Java虚拟机崩溃。这种情况仅凭程序自身是无法处理的，在程序中也不会对Error异常进行捕捉和抛出。

　　异常（Exception）又分为**RuntimeException(运行时异常)和CheckedException(检查时异常)**，两者区别如下：

- RuntimeException：程序**运行过程中**才可能发生的异常。一般为代码的逻辑错误。例如：类型错误转换，数组下标访问越界，空指针异常、找不到指定类等等。

- CheckedException：**编译期间**可以检查到的异常，必须显式的进行处理（捕获或者抛出到上一层）。例如：IOException, FileNotFoundException等等。



## 2.3 异常操作

### 2.3.1 异常创建(异常生成)

#### 2.3.1.1 自定义异常

我们可以通过继承来创建新的异常类。在继承时，我们往往需要重写构造方法。**异常有两个构造方法，一个没有参数，一个有一个String参数**。比如:

```
class BatteryUsageException extends Exception
{
  public BatteryUsageException() {}
  public BatteryUsageException(String msg) {
    super(msg);
  }
}
```

我们可以在衍生类中提供更多异常相关的方法和信息。

在自定义异常时，要小心选择所继承的基类。一个更具体的类要包含更多的异常信息，比如IOException相对于Exception



### 2.3.2 异常处理

异常处理的步骤：

```
try {

}catch (Exception xxxx) {

}finally {

}
```

**catch的括号有一个参数，代表所要捕捉的异常的类型。catch会捕捉相应的类型及其衍生类。catch后面的程序块包含了针对该异常类型所要进行的操作。try所监视的程序块可能抛出不止一种类型的异常，所以一个异常处理器可以有多个catch模块。finally后面的程序块是无论是否发生异常，都要执行的#### **catch的括号有一个参数，代表所要捕捉的异常的类型。catch会捕捉相应的类型及其衍生类。catch后面的程序块包含了针对该异常类型所要进行的操作。try所监视的程序块可能抛出不止一种类型的异常，所以一个异常处理器可以有多个catch模块。finally后面的程序块是无论是否发生异常，都要执行的程序**t的一段代码： 

```
try {  
                        mSocket=new Socket(ip,port);  
                        if(mSocket!=null)  
                        {  
                            Log.i("Client","socket is create");  
                            clientInput=new ClientInputThread(mSocket);  
                            clientOutput=new ClientOutputThread(mSocket);  
                            clientInput.setStart(true);  
                            clientOutput.setStart(true);  
                          
                            clientInput.start();  
                            clientOutput.start();  
                              
                        }  
                        else {  
                            Log.i("Client","socket is not create");  
                        //  Toast.makeText(, "亲，服务器端连接出错",0).show();  
                        }  
                    } catch (UnknownHostException e) {  
                        // TODO Auto-generated catch block  
                        e.printStackTrace();  
                    } catch (IOException e) {  
                        // TODO Auto-generated catch block  
                        e.printStackTrace();  
                          
                    }finally{}   
```

从上述代码可以看到异常处理的步骤为

![img](http://img.blog.csdn.net/20160331170833113)

#### 2.3.2.1 try、catch、finally三个语句块应注意的问题

  第一：try、catch、finally三个语句块均不能单独使用，三者可以组成 try...catch...finally、try...catch、try...finally三种结构，catch语句可以有一个或多个，finally语句最多一个。
  第二：try、catch、finally三个代码块中变量的作用域为代码块内部，分别独立而不能相互访问。如果要在三个块中都可以访问，则需要将变量定义到这些块的外面。
  第三：多个catch块时候，最多只会匹配其中一个异常类且只会执行该catch块代码，而不会再执行其它的catch块，且匹配catch语句的顺序为从上到下，也可能所有的catch都没执行。
   第四：先Catch子类异常再Catch父类异常。
用示意图表示如下：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427162345.jpeg)

#### 2.3.2.2 throw、throws关键字

  throw关键字是用于方法体内部，用来抛出一个Throwable类型的异常。如果抛出了检查异常，则还应该在方法头部声明方法可能抛出的异常类型。该方法的调用者也必须检查处理抛出的异常。如果所有方法都层层上抛获取的异常，最终JVM会进行处理，处理也很简单，就是打印异常消息和堆栈信息。throw关键字用法如下： 

```
public static void test() throws Exception  
{  
   throw new Exception("方法test中的Exception");  
}  
```

  throws关键字用于方法体外部的方法声明部分，用来声明方法可能会抛出某些异常。仅当抛出了检查异常，该方法的调用者才必须处理或者重新抛出该异常。当方法的调用者无力处理该异常的时候，应该继续抛出.

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210427162347.jpeg)

![img](http://img.blog.csdn.net/20160331171522522)

注意一个方法throws出某个异常但是该方法内部可以不throw出该异常，代码如下： 

```
class ER extends RuntimeException  
{  
  
}  
class SomeClass  
{  
    public void fun()throws ER  
    {  
        System.out.println("AAAA");  
          
    }  
}  
  
public class ExceptionTest {  
  
    public static void main(String[] args) {  
        // TODO Auto-generated method stub  
        SomeClass A=new SomeClass();  
        A.fun();  
    }  
}  
```

程序运行结果如下：AAAA。



#### 2.3.2.3 throw 

```java
public void save(User user)
{
 if(user  == null) 
 throw new IllegalArgumentException("User对象为空");
 //......

}
```





#### 2.3.2.4 throws

```java
public void foo() throws ExceptionType1 , ExceptionType2 ,ExceptionTypeN
{ 
 //foo内部可以抛出 ExceptionType1 , ExceptionType2 ,ExceptionTypeN 类的异常，或者他们的子类的异常对象。
}
```



## 2.4 异常调用链





## 2.5 异常打印----关联函数调用栈打印

printStackTrace

![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210611070501.png)



![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210611070516.png)


## 2.6 特殊情况异常处理

### 2.6.1 try-catch-finally 的执行顺序   (finally遇上return)



#### 2.6.1.1 面试题一：

```java
public static void main(String[] args){
    int result = test1();
    System.out.println(result);
}

public static int test1(){
    int i = 1;
    try{
        i++;
        System.out.println("try block, i = "+i);
    }catch(Exception e){
        i--;
        System.out.println("catch block i = "+i);
    }finally{
        i = 10;
        System.out.println("finally block i = "+i);
    }
    return i;
}
```

输出结果

```
try block, i = 2
finally block i = 10
10
```



这算一个相当简单的问题了，没有坑，下面我们稍微改动一下：

```java
public static int test2(){
    int i = 1;
    try{
        i++;
        throw new Exception();
    }catch(Exception e){
        i--;
        System.out.println("catch block i = "+i);
    }finally{
        i = 10;
        System.out.println("finally block i = "+i);
    }
    return i;
}
```

输出结果如下：

```
catch block i = 1
finally block i = 10
10
```

运行结果想必也是意料之中吧，程序抛出一个异常，然后被本方法的 catch 块捕获并进行了处理。



#### 2.6.1.2 面试题二：

```java
public static void main(String[] args){
    int result = test3();
    System.out.println(result);
}

public static int test3(){
    //try 语句块中有 return 语句时的整体执行顺序
    int i = 1;
    try{
        i++;
        System.out.println("try block, i = "+i);
        return i;
    }catch(Exception e){
        i ++;
        System.out.println("catch block i = "+i);
        return i;
    }finally{
        i = 10;
        System.out.println("finally block i = "+i);
    }
}
```



输出结果如下：

```
try block, i = 2
finally block i = 10
2
```

是不是有点疑惑？明明我 try 语句块中有 return 语句，可为什么最终还是执行了 finally 块中的代码？

我们反编译这个类，看看这个 test3 方法编译后的字节码的实现：

```
0: iconst_1         //将 1 加载进操作数栈
1: istore_0         //将操作数栈 0 位置的元素存进局部变量表
2: iinc          0, 1   //将局部变量表 0 位置的元素直接加一（i=2）
5: getstatic     #3     // 5-27 行执行的 println 方法                
8: new           #5                  
11: dup
12: invokespecial #6                                                     
15: ldc           #7 
17: invokevirtual #8                                                     
20: iload_0         
21: invokevirtual #9                                                     24: invokevirtual #10                
27: invokevirtual #11                 
30: iload_0         //将局部变量表 0 位置的元素加载进操作栈（2）
31: istore_1        //把操作栈顶的元素存入局部变量表位置 1 处
32: bipush        10 //加载一个常量到操作栈（10）
34: istore_0        //将 10 存入局部变量表 0 处
35: getstatic     #3  //35-57 行执行 finally中的println方法             
38: new           #5                  
41: dup
42: invokespecial #6                  
45: ldc           #12                 
47: invokevirtual #8                  
50: iload_0
51: invokevirtual #9                
54: invokevirtual #10                 
57: invokevirtual #11                 
60: iload_1         //将局部变量表 1 位置的元素加载进操作栈（2）
61: ireturn         //将操作栈顶元素返回（2）
-------------------try + finally 结束 ------------
------------------下面是 catch + finally，类似的 ------------
62: astore_1
63: iinc          0, 1
.......
.......
```

从我们的分析中可以看出来，finally 代码块中的内容始终会被执行，无论程序是否出现异常的原因就是，编译器会将 finally 块中的代码复制两份并分别添加在 try 和 catch 的后面。

可能有人会所疑惑，原本我们的 i 就被存储在局部变量表 0 位置，而最后 finally 中的代码也的确将 slot 0 位置填充了数值 10，可为什么最后程序依然返回的数值 2 呢？

仔细看字节码，你会发现在 return 语句返回之前，虚拟机会将待返回的值压入操作数栈，等待返回，即使 finally 语句块对 i 进行了修改，但是待返回的值已经确实的存在于操作数栈中了，所以不会影响程序返回结果。

#### 2.6.1.3 面试题三

```
public static int test4(){
    //finally 语句块中有 return 语句
    int i = 1;
    try{
        i++;
        System.out.println("try block, i = "+i);
        return i;
    }catch(Exception e){
        i++;
        System.out.println("catch block i = "+i);
        return i;
    }finally{
        i++;
        System.out.println("finally block i = "+i);
        return i;
    }
}
```

运行结果：

```
try block, i = 2
finally block i = 3
3
```

其实你从它的字节码指令去看整个过程，而不要单单四记它的执行过程。

![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210615160539.png)

你会发现程序最终会采用 finally 代码块中的 return 语句进行返回，而直接忽略 try 语句块中的 return 指令。

最后，对于异常的使用有一个不成文的约定：**尽量在某个集中的位置进行统一处理，不要到处的使用 try-catch，否则会使得代码结构混乱不堪**。



#### 2.6.1.4 注意点

### 2.6.2 当finally遇上return

首先一个不容易理解的事实：

在 try块中即便有return，break，continue等改变执行流的语句，finally也会执行。

```java
public static void main(String[] args)
{
 int re = bar();
 System.out.println(re);
}
private static int bar() 
{
 try{
 return 5;
 } finally{
 System.out.println("finally");
 }
}
/*输出：
finally
*/
```

很多人面对这个问题时，总是在归纳执行的顺序和规律，不过我觉得还是很难理解。我自己总结了一个方法。用如下GIF图说明。

也就是说：try…catch…finally中的return 只要能执行，就都执行了，他们共同向同一个内存地址（假设地址是0×80）写入返回值，**后执行的将覆盖先执行的数据，而真正被调用者取的返回值就是最后一次写入的**。那么，按照这个思想，下面的这个例子也就不难理解了。

finally中的return 会覆盖 try 或者catch中的返回值。

```java
public static void main(String[] args)
 {
 int result;

        result  =  foo();
 System.out.println(result); /////////2

        result = bar();
 System.out.println(result); /////////2
 }

 @SuppressWarnings("finally")
 public static int foo()
 {
        trz{
 int a = 5 / 0;
 } catch (Exception e){
 return 1;
 } finally{
 return 2;
 }

 }

 @SuppressWarnings("finally")
 public static int bar()
 {
 try {
 return 1;
 }finally {
 return 2;
 }
 }
```



finally中的return会抑制（消灭）前面try或者catch块中的异常

```java
class TestException
{
 public static void main(String[] args)
 {
 int result;
 try{
            result = foo();
 System.out.println(result); //输出100
 } catch (Exception e){
 System.out.println(e.getMessage()); //没有捕获到异常
 }

 try{
            result  = bar();
 System.out.println(result); //输出100
 } catch (Exception e){
 System.out.println(e.getMessage()); //没有捕获到异常
 }
 }

 //catch中的异常被抑制
 @SuppressWarnings("finally")
 public static int foo() throws Exception
 {
 try {
 int a = 5/0;
 return 1;
 }catch(ArithmeticException amExp) {
 throw new Exception("我将被忽略，因为下面的finally中使用了return");
 }finally {
 return 100;
 }
 }

 //try中的异常被抑制
 @SuppressWarnings("finally")
 public static int bar() throws Exception
 {
 try {
 int a = 5/0;
 return 1;
 }finally {
 return 100;
 }
 }
}
```



finally中的异常会覆盖（消灭）前面try或者catch中的异常

```java
class TestException
{
 public static void main(String[] args)
 {
 int result;
 try{
            result = foo();
 } catch (Exception e){
 System.out.println(e.getMessage()); //输出：我是finaly中的Exception
 }

 try{
            result  = bar();
 } catch (Exception e){
 System.out.println(e.getMessage()); //输出：我是finaly中的Exception
 }
 }

 //catch中的异常被抑制
 @SuppressWarnings("finally")
 public static int foo() throws Exception
 {
 try {
 int a = 5/0;
 return 1;
 }catch(ArithmeticException amExp) {
 throw new Exception("我将被忽略，因为下面的finally中抛出了新的异常");
 }finally {
 throw new Exception("我是finaly中的Exception");
 }
 }

 //try中的异常被抑制
 @SuppressWarnings("finally")
 public static int bar() throws Exception
 {
 try {
 int a = 5/0;
 return 1;
 }finally {
 throw new Exception("我是finaly中的Exception");
 }

 }
}
```




## 2.7 线程

Java的异常执行流程是线程独立的，线程之间没有影响

try { } catch()无法捕获到非自己线程的异常


## 2.8 注意点

- **try块中的局部变量和catch块中的局部变量（包括异常变量），以及finally中的局部变量，他们之间不可共享使用。**
- 每一个catch块用于处理一个异常。异常匹配是按照**catch块的顺序从上往下寻找**的，只有第一个匹配的catch会得到执行。匹配时，不仅运行精确匹配，也支持父类匹配，因此，如果同一个try块下的多个catch异常类型有父子关系，应该将子类异常放在前面，父类异常放在后面，这样保证每个catch块都有存在的意义。
- finally块不管异常是否发生，只要对应的try执行了，则它一定也执行。**只有一种方法让finally块不执行：System.exit()**。因此finally块通常用来做资源释放操作：关闭文件，关闭数据库连接等等。良好的编程习惯是：在try块中打开资源，在finally块中清理释放这些资源。
- finally块没有处理异常的能力。处理异常的只能是catch块



# 3 深入理解异常-异常实现原理

我们再深入理解下异常，看下底层实现。

## 3.1 JVM处理异常的机制？

提到JVM处理异常的机制，就需要提及Exception Table，以下称为异常表。我们暂且不急于介绍异常表，先看一个简单的 Java 处理异常的小例子。

```java
public static void simpleTryCatch() {
   try {
       testNPE();
   } catch (Exception e) {
       e.printStackTrace();
   }
}
   
```

上面的代码是一个很简单的例子，用来捕获处理一个潜在的空指针异常。

当然如果只是看简简单单的代码，我们很难看出什么高深之处，更没有了今天文章要谈论的内容。

所以这里我们需要借助一把神兵利器，它就是javap,一个用来拆解class文件的工具，和javac一样由JDK提供。

然后我们使用javap来分析这段代码（需要先使用javac编译）

```java
//javap -c Main
 public static void simpleTryCatch();
    Code:
       0: invokestatic  #3                  // Method testNPE:()V
       3: goto          11
       6: astore_0
       7: aload_0
       8: invokevirtual #5                  // Method java/lang/Exception.printStackTrace:()V
      11: return
    Exception table:
       from    to  target type
           0     3     6   Class java/lang/Exception
   
```


看到上面的代码，应该会有会心一笑，因为终于看到了Exception table，也就是我们要研究的异常表。

异常表中包含了一个或多个异常处理者(Exception Handler)的信息，这些信息包含如下

- **from** 可能发生异常的起始点
- **to** 可能发生异常的结束点
- **target** 上述from和to之前发生异常后的异常处理者的位置
- **type** 异常处理者处理的异常的类信息

**那么异常表用在什么时候呢**

答案是异常发生的时候，当一个异常发生时

- 1.JVM会在当前出现异常的方法中，查找异常表，是否有合适的处理者来处理
- 2.如果当前方法异常表不为空，并且异常符合处理者的from和to节点，并且type也匹配，则JVM调用位于target的调用者来处理。
- 3.如果上一条未找到合理的处理者，则继续查找异常表中的剩余条目
- 4.如果当前方法的异常表无法处理，则向上查找（弹栈处理）刚刚调用该方法的调用处，并重复上面的操作。
- 5.如果所有的栈帧被弹出，仍然没有处理，则抛给当前的Thread，Thread则会终止。
- 6.如果当前Thread为最后一个非守护线程，且未处理异常，则会导致JVM终止运行。

以上就是JVM处理异常的一些机制。

**try catch -finally**

除了简单的try-catch外，我们还常常和finally做结合使用。比如这样的代码

```java
public static void simpleTryCatchFinally() {
   try {
       testNPE();
   } catch (Exception e) {
       e.printStackTrace();
   } finally {
       System.out.println("Finally");
   }
}
    
```


同样我们使用javap分析一下代码

```java
public static void simpleTryCatchFinally();
    Code:
       0: invokestatic  #3                  // Method testNPE:()V
       3: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
       6: ldc           #7                  // String Finally
       8: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      11: goto          41
      14: astore_0
      15: aload_0
      16: invokevirtual #5                  // Method java/lang/Exception.printStackTrace:()V
      19: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
      22: ldc           #7                  // String Finally
      24: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      27: goto          41
      30: astore_1
      31: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
      34: ldc           #7                  // String Finally
      36: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      39: aload_1
      40: athrow
      41: return
    Exception table:
       from    to  target type
           0     3    14   Class java/lang/Exception
           0     3    30   any
          14    19    30   any
    
```

和之前有所不同，这次异常表中，有三条数据，而我们仅仅捕获了一个Exception, 异常表的后两个item的type为any; 上面的三条异常表item的意思为:

- 如果0到3之间，发生了Exception类型的异常，调用14位置的异常处理者。
- 如果0到3之间，无论发生什么异常，都调用30位置的处理者
- 如果14到19之间（即catch部分），不论发生什么异常，都调用30位置的处理者。

再次分析上面的Java代码，finally里面的部分已经被提取到了try部分和catch部分。我们再次调一下代码来看一下

```java
public static void simpleTryCatchFinally();
    Code:
      //try 部分提取finally代码，如果没有异常发生，则执行输出finally操作，直至goto到41位置，执行返回操作。  

       0: invokestatic  #3                  // Method testNPE:()V
       3: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
       6: ldc           #7                  // String Finally
       8: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      11: goto          41

      //catch部分提取finally代码，如果没有异常发生，则执行输出finally操作，直至执行got到41位置，执行返回操作。
      14: astore_0
      15: aload_0
      16: invokevirtual #5                  // Method java/lang/Exception.printStackTrace:()V
      19: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
      22: ldc           #7                  // String Finally
      24: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      27: goto          41
      //finally部分的代码如果被调用，有可能是try部分，也有可能是catch部分发生异常。
      30: astore_1
      31: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
      34: ldc           #7                  // String Finally
      36: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      39: aload_1
      40: athrow     //如果异常没有被catch捕获，而是到了这里，执行完finally的语句后，仍然要把这个异常抛出去，传递给调用处。
      41: return
    
```


**Catch先后顺序的问题**

我们在代码中的catch的顺序决定了异常处理者在异常表的位置，所以，越是具体的异常要先处理，否则就会出现下面的问题

```java
private static void misuseCatchException() {
   try {
       testNPE();
   } catch (Throwable t) {
       t.printStackTrace();
   } catch (Exception e) { //error occurs during compilings with tips Exception Java.lang.Exception has already benn caught.
       e.printStackTrace();
   }
}
    
```


这段代码会导致编译失败，因为先捕获Throwable后捕获Exception，会导致后面的catch永远无法被执行。

**Return 和finally的问题**

这算是我们扩展的一个相对比较极端的问题，就是类似这样的代码，既有return，又有finally，那么finally导致会不会执行

```java
public static String tryCatchReturn() {
   try {
       testNPE();
       return  "OK";
   } catch (Exception e) {
       return "ERROR";
   } finally {
       System.out.println("tryCatchReturn");
   }
}
    
```


答案是finally会执行，那么还是使用上面的方法，我们来看一下为什么finally会执行。

```java
public static java.lang.String tryCatchReturn();
    Code:
       0: invokestatic  #3                  // Method testNPE:()V
       3: ldc           #6                  // String OK
       5: astore_0
       6: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: ldc           #8                  // String tryCatchReturn
      11: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      14: aload_0
      15: areturn       返回OK字符串，areturn意思为return a reference from a method
      16: astore_0
      17: ldc           #10                 // String ERROR
      19: astore_1
      20: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      23: ldc           #8                  // String tryCatchReturn
      25: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      28: aload_1
      29: areturn  //返回ERROR字符串
      30: astore_2
      31: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      34: ldc           #8                  // String tryCatchReturn
      36: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      39: aload_2
      40: athrow  如果catch有未处理的异常，抛出去。
    
```


## 3.2 异常是否耗时？为什么会耗时？

说用异常慢，首先来看看异常慢在哪里？有多慢？下面的测试用例简单的测试了建立对象、建立异常对象、抛出并接住异常对象三者的耗时对比：

```java
public class ExceptionTest {  
  
    private int testTimes;  
  
    public ExceptionTest(int testTimes) {  
        this.testTimes = testTimes;  
    }  
  
    public void newObject() {  
        long l = System.nanoTime();  
        for (int i = 0; i < testTimes; i++) {  
            new Object();  
        }  
        System.out.println("建立对象：" + (System.nanoTime() - l));  
    }  
  
    public void newException() {  
        long l = System.nanoTime();  
        for (int i = 0; i < testTimes; i++) {  
            new Exception();  
        }  
        System.out.println("建立异常对象：" + (System.nanoTime() - l));  
    }  
  
    public void catchException() {  
        long l = System.nanoTime();  
        for (int i = 0; i < testTimes; i++) {  
            try {  
                throw new Exception();  
            } catch (Exception e) {  
            }  
        }  
        System.out.println("建立、抛出并接住异常对象：" + (System.nanoTime() - l));  
    }  
  
    public static void main(String[] args) {  
        ExceptionTest test = new ExceptionTest(10000);  
        test.newObject();  
        test.newException();  
        test.catchException();  
    }  
}      
```

运行结果：
```java
建立对象：575817  
建立异常对象：9589080  
建立、抛出并接住异常对象：47394475  

```


# 4 Android   [[Android中Exception处理逻辑]]


> Android 8.0中，Thread类增加了一个接口叫setUncaughtExceptionPreHandler，它会注册在分发异常处理时的回调，用于Android系统（平台）使用。
>
> Thread的dispatchUncaughtException负责处理异常的分发逻辑。
>
> Android中，异常分发顺序为：
> 首先处理setUncaughtExceptionPreHandler注册的异常处理方法；
> 然后处理线程私有的（uncaughtExceptionHandler）Handler异常处理方法；
> 如果私有Handler不存在，则处理ThreadGroup的Handler异常处理方法；
>
> ThreadGroup中，优先调用父线程组的处理逻辑，否则，调用通过setUncaughtExceptionHandler方法注册异常处理Handler。
>
> 系统默认异常处理逻辑在KillApplicationHandler类的uncaughtException方法中，系统默认Crash弹框等逻辑是通过AMS的handleApplicationCrash方法执行的。

## 4.1 总结

本文主要以源码的视角，详细介绍了到应用crash后系统的处理流程：

1. 首先发生crash所在进程，在创建之初便准备好了defaultUncaughtHandler，用来来处理Uncaught Exception，并输出当前crash基本信息；
2. 调用当前进程中的AMP.handleApplicationCrash；经过binder ipc机制，传递到system_server进程；
3. 接下来，进入system_server进程，调用binder服务端执行AMS.handleApplicationCrash；
4. 从`mProcessNames`查找到目标进程的ProcessRecord对象；并将进程crash信息输出到目录`/data/system/dropbox`；
5. 执行makeAppCrashingLocked
   - 创建当前用户下的crash应用的error receiver，并忽略当前应用的广播；
   - 停止当前进程中所有activity中的WMS的冻结屏幕消息，并执行相关一些屏幕相关操作；
6. 再执行handleAppCrashLocked方法，
   - 当1分钟内同一进程`连续crash两次`时，且`非persistent`进程，则直接结束该应用所有activity，并杀死该进程以及同一个进程组下的所有进程。然后再恢复栈顶第一个非finishing状态的activity;
   - 当1分钟内同一进程`连续crash两次`时，且`persistent`进程，，则只执行恢复栈顶第一个非finishing状态的activity;
   - 当1分钟内同一进程`未发生连续crash两次`时，则执行结束栈顶正在运行activity的流程。
7. 通过mUiHandler发送消息`SHOW_ERROR_MSG`，弹出crash对话框；
8. 到此，system_server进程执行完成。回到crash进程开始执行杀掉当前进程的操作；
9. 当crash进程被杀，通过binder死亡通知，告知system_server进程来执行appDiedLocked()；
10. 最后，执行清理应用相关的activity/service/ContentProvider/receiver组件信息。

这基本就是整个应用Crash后系统的执行过程。 最后，再说说对于同一个app连续crash的情况：

- 当60s内连续crash两次的非persistent进程时，被认定为bad进程：那么如果第3次从后台启动该进程(Intent.getFlags来判断)，则会拒绝创建进程；
- 当crash次数达到两次的非persistent进程时，则再次杀该进程，即便允许自启的service也会在被杀后拒绝再次启动



# 5 面试题

## 5.1 异常处理完成以后，Exception对象会发生什么变化？

Exception对象会在下一个垃圾回收过程中被回收掉。

## 5.2 扩展：错误和异常的区别(Error vs Exception)

1. java.lang.Error: Throwable 的子类，用于标记严重错误。合理的应用程序不应该去 try/catch 这种错误。绝大多数的错误都是非正常的，就根本不该出现的。

java.lang.Exception: Throwable 的子类，用于指示一种合理的程序想去 catch 的条件。即它仅仅是一种程序运行条件，而非严重错误，并且鼓励用户程序去 catch 它。

1. Error 和 RuntimeException 及其子类都是未检查的异常（unchecked exceptions），而所有其他的 Exception 类都是检查了的异常（checked exceptions）

 

## 5.3 题目一  return

```
package defineexception;
public class ExceptionTest3
{
    public void method()
    {
        try
        {
            System.out.println("try");
            return;
        }
        catch(Exception ex)
        {
            System.out.println("异常发生了");
        }
        finally
        {
            System.out.println("finally");
        }
        System.out.println("异常处理后续的代码");
    }
    public static void main(String[] args)
    {
        ExceptionTest3 test =new ExceptionTest3();
        test.method();
    }
}
```

### 5.3.1 结果：

```
try
finally
```

### 5.3.2 分析

> **try块中存在return语句，那么首先也需要将finally块中的代码执行完毕，再执行return语句，而且之后的其他代码也不会再执行了**

## 5.4 题目二 exit

```
package defineexception;
public class ExceptionTest3
{
    public void method()
    {
        try
        {
            System.out.println("try");
            System.exit(0);
        }
        catch(Exception ex)
        {
            System.out.println("异常发生了");
        }
        finally
        {
            System.out.println("finally");
        }
        System.out.println("异常处理后续的代码");
    }
    public static void main(String[] args)
    {
        ExceptionTest3 test =new ExceptionTest3();
        test.method();
    }
}
```

### 5.4.1 结果

```
try
```

### 5.4.2 分析

> #### **先执行try块中的`System.exit(0)`语句，已经退出了虚拟机系统，所以不会执行`finally`块的代码**



## 5.5 Error 和 Exception 区别是什么？

Java 中，所有的异常都有一个共同的祖先 `java.lang` 包中的 `Throwable` 类。`Throwable` 类有两个重要的子类 `Exception`（异常）和 `Error`（错误）。

`Exception` 和 `Error` 二者都是 Java 异常处理的重要子类，各自都包含大量子类。

-   **`Exception`** :程序本身可以处理的异常，可以通过 `catch` 来进行捕获，通常遇到这种错误，应对其进行处理，使应用程序可以继续正常运行。`Exception` 又可以分为运行时异常(RuntimeException, 又叫非受检查异常)和非运行时异常(又叫受检查异常) 。
-   **`Error`** ：`Error` 属于程序无法处理的错误 ，我们没办法通过 `catch` 来进行捕获 。例如，系统崩溃，内存不足，堆栈溢出等，编译器不会对这类错误进行检测，一旦这类错误发生，通常应用程序会被终止，仅靠应用程序本身无法恢复。

[![](https://camo.githubusercontent.com/3b5dcd99938a078ae39c879ab3bbdf5049a5cfc1b76c69f0d7d20b977a5ea3eb/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f696d6167652d32303231303232373130333235363233342e706e67)](https://camo.githubusercontent.com/3b5dcd99938a078ae39c879ab3bbdf5049a5cfc1b76c69f0d7d20b977a5ea3eb/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f696d6167652d32303231303232373130333235363233342e706e67)

## 5.6 非受检查异常(运行时异常)和受检查异常(一般异常)区别是什么？

非受检查异常：包括 `RuntimeException` 类及其子类，表示 JVM 在运行期间可能出现的异常。 Java 编译器不会检查运行时异常。例如：`NullPointException(空指针)`、`NumberFormatException（字符串转换为数字）`、`IndexOutOfBoundsException(数组越界)`、`ClassCastException(类转换异常)`、`ArrayStoreException(数据存储异常，操作数组时类型不一致)`等。

受检查异常：是Exception 中除 `RuntimeException` 及其子类之外的异常。 Java 编译器会检查受检查异常。常见的受检查异常有： IO 相关的异常、`ClassNotFoundException` 、`SQLException`等。

**非受检查异常和受检查异常之间的区别**：是否强制要求调用者必须处理此异常，如果强制要求调用者必须进行处理，那么就使用受检查异常，否则就选择非受检查异常。

## 5.7 throw 和 throws 的区别是什么？

Java 中的异常处理除了包括捕获异常和处理异常之外，还包括声明异常和拋出异常，可以通过 throws 关键字在方法上声明该方法要拋出的异常，或者在方法内部通过 throw 拋出异常对象。

throws 关键字和 throw 关键字在使用上的几点区别如下：

-   throw 关键字用在方法内部，只能用于抛出一种异常，用来抛出方法或代码块中的异常，受查异常和非受查异常都可以被抛出。
-   throws 关键字用在方法声明上，可以抛出多个异常，用来标识该方法可能抛出的异常列表。一个方法用 throws 标识了可能抛出的异常列表，调用该方法的方法中必须包含可处理异常的代码，否则也要在方法签名中用 throws 关键字声明相应的异常。

举例如下：

**throw 关键字**：

public static void main(String[] args) {
		String s = "abc";
		if(s.equals("abc")) {
			throw new NumberFormatException();
		} else {
			System.out.println(s);
		}
		//function();
}

**throws 关键字**：

public static void function() throws NumberFormatException{
		String s = "abc";
		System.out.println(Double.parseDouble(s));
	}
	
	public static void main(String[] args) {
		try {
			function();
		} catch (NumberFormatException e) {
			System.err.println("非数据类型不能转换。");
			//e.printStackTrace();
		}
}


## 5.8 final、finally、finalize 有什么区别？
-   final可以修饰类、变量、方法，修饰类表示该类不能被继承、修饰方法表示该方法不能被重写、修饰变量表示该变量是一个常量不能被重新赋值。
-   finally一般作用在try-catch代码块中，在处理异常的时候，通常我们将一定要执行的代码方法finally代码块中，表示不管是否出现异常，该代码块都会执行，一般用来存放一些关闭资源的代码。
-   finalize是一个方法，属于Object类的一个方法，而Object类是所有类的父类，该方法一般由垃圾回收器来调用，当我们调用System的gc()方法的时候，由垃圾回收器调用finalize(),回收垃圾。



## 5.9 NoClassDefFoundError 和 ClassNotFoundException 区别？

NoClassDefFoundError 是一个 Error 类型的异常，是由 JVM 引起的，不应该尝试捕获这个异常。引起该异常的原因是 JVM 或 ClassLoader 尝试加载某类时在内存中找不到该类的定义，**该动作发生在运行期间，即编译时该类存在，但是在运行时却找不到了，可能是编译后被删除了等原因导致。**

ClassNotFoundException 是一个受检查异常，需要显式地使用 try-catch 对其进行捕获和处理，或在方法签名中用 throws 关键字进行声明。**当使用 Class.forName, ClassLoader.loadClass 或 ClassLoader.findSystemClass 动态加载类到内存的时候，通过传入的类路径参数没有找到该类，就会抛出该异常；另一种抛出该异常的可能原因是某个类已经由一个类加载器加载至内存中，另一个加载器又尝试去加载它**。

## 5.10 Java常见异常有哪些？

-   java.lang.IllegalAccessError：违法访问错误。当一个应用试图访问、修改某个类的域（Field）或者调用其方法，但是又违反域或方法的可见性声明，则抛出该异常。
-   java.lang.InstantiationError：实例化错误。当一个应用试图通过Java的new操作符构造一个抽象类或者接口时抛出该异常.
-   java.lang.OutOfMemoryError：内存不足错误。当可用内存不足以让Java虚拟机分配给一个对象时抛出该错误。
-   java.lang.StackOverflowError：堆栈溢出错误。当一个应用递归调用的层次太深而导致堆栈溢出或者陷入死循环时抛出该错误。
-   java.lang.ClassCastException：类造型异常。假设有类A和B（A不是B的父类或子类），O是A的实例，那么当强制将O构造为类B的实例时抛出该异常。该异常经常被称为强制类型转换异常。
-   java.lang.ClassNotFoundException：找不到类异常。当应用试图根据字符串形式的类名构造类，而在遍历CLASSPAH之后找不到对应名称的class文件时，抛出该异常。
-   java.lang.ArithmeticException：算术条件异常。譬如：整数除零等。
-   java.lang.ArrayIndexOutOfBoundsException：数组索引越界异常。当对数组的索引值为负数或大于等于数组大小时抛出。
-   java.lang.IndexOutOfBoundsException：索引越界异常。当访问某个序列的索引值小于0或大于等于序列大小时，抛出该异常。
-   java.lang.InstantiationException：实例化异常。当试图通过newInstance()方法创建某个类的实例，而该类是一个抽象类或接口时，抛出该异常。
-   java.lang.NoSuchFieldException：属性不存在异常。当访问某个类的不存在的属性时抛出该异常。
-   java.lang.NoSuchMethodException：方法不存在异常。当访问某个类的不存在的方法时抛出该异常。
-   java.lang.NullPointerException：空指针异常。当应用试图在要求使用对象的地方使用了null时，抛出该异常。譬如：调用null对象的实例方法、访问null对象的属性、计算null对象的长度、使用throw语句抛出null等等。
-   java.lang.NumberFormatException：数字格式异常。当试图将一个String转换为指定的数字类型，而该字符串确不满足数字类型要求的格式时，抛出该异常。
-   java.lang.StringIndexOutOfBoundsException：字符串索引越界异常。当使用索引值访问某个字符串中的字符，而该索引值小于0或大于等于序列大小时，抛出该异常。

## 5.11 try-catch-finally 中哪个部分可以省略？

catch 可以省略。更为严格的说法其实是：try只适合处理运行时异常，try+catch适合处理运行时异常+普通异常。也就是说，如果你只用try去处理普通异常却不加以catch处理，编译是通不过的，因为编译器硬性规定，普通异常如果选择捕获，则必须用catch显示声明以便进一步处理。而运行时异常在编译时没有如此规定，所以catch可以省略，你加上catch编译器也觉得无可厚非。

理论上，编译器看任何代码都不顺眼，都觉得可能有潜在的问题，所以你即使对所有代码加上try，代码在运行期时也只不过是在正常运行的基础上加一层皮。但是你一旦对一段代码加上try，就等于显示地承诺编译器，对这段代码可能抛出的异常进行捕获而非向上抛出处理。如果是普通异常，编译器要求必须用catch捕获以便进一步处理；如果运行时异常，捕获然后丢弃并且+finally扫尾处理，或者加上catch捕获以便进一步处理。

至于加上finally，则是在不管有没捕获异常，都要进行的“扫尾”处理。

## 5.12 try-catch-finally 中，如果 catch 中 return 了，finally 还会执行吗？

会执行，在 return 前执行。

在 finally 中改变返回值的做法是不好的，因为如果存在 finally 代码块，try中的 return 语句不会立马返回调用者，而是记录下返回值待 finally 代码块执行完毕之后再向调用者返回其值，然后如果在 finally 中修改了返回值，就会返回修改后的值。显然，在 finally 中返回或者修改返回值会对程序造成很大的困扰，Java 中也可以通过提升编译器的语法检查级别来产生警告或错误。 **代码示例1：**

public static int getInt() {
    int a = 10;
    try {
        System.out.println(a / 0);
        a = 20;
    } catch (ArithmeticException e) {
        a = 30;
        return a;
        /*
 * return a 在程序执行到这一步的时候，这里不是return a 而是 return 30；这个返回路径就形成了
 * 但是呢，它发现后面还有finally，所以继续执行finally的内容，a=40
 * 再次回到以前的路径,继续走return 30，形成返回路径之后，这里的a就不是a变量了，而是常量30
 */
    } finally {
        a = 40;
    }
	return a;
 }

//执行结果：30

**代码示例2：**

public static int getInt() {
    int a = 10;
    try {
        System.out.println(a / 0);
        a = 20;
    } catch (ArithmeticException e) {
        a = 30;
        return a;
    } finally {
        a = 40;
        //如果这样，就又重新形成了一条返回路径，由于只能通过1个return返回，所以这里直接返回40
        return a; 
    }

}

// 执行结果：40



## 5.13 Excption与Error包结构

Java可抛出(Throwable)的结构分为三种类型：**被检查的异常(CheckedException)**，**运行时异常(RuntimeException)**，**错误(Error)**。

### 5.13.1 1、运行时异常

**定义：**RuntimeException及其子类都被称为运行时异常。

**特点：**Java编译器不会检查它。也就是说，当程序中可能出现这类异常时，倘若既"没有通过throws声明抛出它"，也"没有用try-catch语句捕获它"，还是会编译通过。例如，除数为零时产生的ArithmeticException异常，数组越界时产生的IndexOutOfBoundsException异常，fail-fast机制产生的ConcurrentModificationException异常（java.util包下面的所有的集合类都是快速失败的，“快速失败”也就是fail-fast，它是Java集合的一种错误检测机制。当多个线程对集合进行结构上的改变的操作时，有可能会产生fail-fast机制。记住是有可能，而不是一定。例如：假设存在两个线程（线程1、线程2），线程1通过Iterator在遍历集合A中的元素，在某个时候线程2修改了集合A的结构（是结构上面的修改，而不是简单的修改集合元素的内容），那么这个时候程序就会抛出 ConcurrentModificationException 异常，从而产生fail-fast机制，这个错叫并发修改异常。Fail-safe，java.util.concurrent包下面的所有的类都是安全失败的，在遍历过程中，如果已经遍历的数组上的内容变化了，迭代器不会抛出ConcurrentModificationException异常。如果未遍历的数组上的内容发生了变化，则有可能反映到迭代过程中。这就是ConcurrentHashMap迭代器弱一致的表现。ConcurrentHashMap的弱一致性主要是为了提升效率，是一致性与效率之间的一种权衡。要成为强一致性，就得到处使用锁，甚至是全局锁，这就与Hashtable和同步的HashMap一样了。）等，都属于运行时异常。

**常见的五种运行时异常：**

- `ClassCastException`（类转换异常）
- `IndexOutOfBoundsException`（数组越界）
- `NullPointerException`（空指针异常）
- `ArrayStoreException`（数据存储异常，操作数组是类型不一致）
- `BufferOverflowException`

### 5.13.2 2、被检查异常

**定义：**Exception类本身，以及Exception的子类中除了"运行时异常"之外的其它子类都属于被检查异常。

**特点 ：** Java编译器会检查它。 此类异常，要么通过throws进行声明抛出，要么通过try-catch进行捕获处理，否则不能通过编译。例如，CloneNotSupportedException就属于被检查异常。

当通过clone()接口去克隆一个对象，而该对象对应的类没有实现Cloneable接口，就会抛出CloneNotSupportedException异常。被检查异常通常都是可以恢复的。 如：

```
IOException
FileNotFoundException
SQLException
```

被检查的异常适用于那些不是因程序引起的错误情况，比如：读取文件时文件不存在引发的`FileNotFoundException`。然而，不被检查的异常通常都是由于糟糕的编程引起的，比如：在对象引用时没有确保对象非空而引起的`NullPointerException`。

**3、错误**

定义 : Error类及其子类。

特点 : 和运行时异常一样，编译器也不会对错误进行检查。

当资源不足、约束失败、或是其它程序无法继续运行的条件发生时，就产生错误。程序本身无法修复这些错误的。例如，VirtualMachineError就属于错误。出现这种错误会导致程序终止运行。OutOfMemoryError、ThreadDeath。

Java虚拟机规范规定JVM的内存分为了好几块，比如堆，栈，程序计数器，方法区等

## 5.14 try {}里有一个return语句，那么紧跟在这个try后的finally{}里的code会不会被执行，什么时候被执行，在return前还是后?

我们知道finally{}中的语句是一定会执行的，那么这个可能正常脱口而出就是return之前，return之后可能就出了这个方法了，鬼知道跑哪里去了，但更准确的应该是在return中间执行，请看下面程序代码的运行结果：

```java
public classTest {
    public static void main(String[]args) {
       System.out.println(newTest().test());;
    }
    static int test()
    {
       intx = 1;
       try
       {
          return x;
       }
       finally
       {
          ++x;
       }
    }
}
1.2.3.4.5.6.7.8.9.10.11.12.13.14.15.16.17.
```

执行结果如下：

```html
1
1.
```

运行结果是1，为什么呢？主函数调用子函数并得到结果的过程，好比主函数准备一个空罐子，当子函数要返回结果时，先把结果放在罐子里，然后再将程序逻辑返回到主函数。所谓返回，就是子函数说，我不运行了，你主函数继续运行吧，这没什么结果可言，结果是在说这话之前放进罐子里的。

## 5.15 运行时异常与一般异常有何异同？

异常表示程序运行过程中可能出现的非正常状态，运行时异常表示虚拟机的通常操作中可能遇到的异常，是一种常见运行错误。java编译器要求方法必须声明抛出可能发生的非运行时异常，但是并不要求必须声明抛出未被捕获的运行时异常。

## 5.16 error和exception有什么区别?

error 表示恢复不是不可能但很困难的情况下的一种严重问题。比如说内存溢出。不可能指望程序能处理这样的情况。exception表示一种设计或实现问题。也就是说，它表示如果程序运行正常，从不会发生的情况。

## 5.17 简单说说Java中的异常处理机制的简单原理和应用

**异常是指java程序运行时（非编译）所发生的非正常情况或错误**，与现实生活中的事件很相似，现实生活中的事件可以包含事件发生的时间、地点、人物、情节等信息，可以用一个对象来表示，Java使用面向对象的方式来处理异常，它把程序中发生的每个异常也都分别封装到一个对象来表示的，该对象中包含有异常的信息。

Java对异常进行了分类，不同类型的异常分别用不同的Java类表示，所有异常的根类为java.lang.Throwable。

Throwable下面又派生了两个子类：

- Error和Exception，Error表示应用程序本身无法克服和恢复的一种严重问题，程序只有奔溃了，例如，说内存溢出和线程死锁等系统问题。
- Exception表示程序还能够克服和恢复的问题，其中又分为系统异常和普通异常：

系统异常是软件本身缺陷所导致的问题，也就是软件开发人员考虑不周所导致的问题，软件使用者无法克服和恢复这种问题，但在这种问题下还可以让软件系统继续运行或者让软件挂掉，例如，数组脚本越界（ArrayIndexOutOfBoundsException），空指针异常（NullPointerException）、类转换异常（ClassCastException）；

普通异常是运行环境的变化或异常所导致的问题，是用户能够克服的问题，例如，网络断线，硬盘空间不够，发生这样的异常后，程序不应该死掉。

java为系统异常和普通异常提供了不同的解决方案，编译器强制普通异常必须try…catch处理或用throws声明继续抛给上层调用方法处理，所以普通异常也称为checked异常，而系统异常可以处理也可以不处理，所以，编译器不强制用try…catch处理或用throws声明，所以系统异常也称为unchecked异常。




## 5.18 JVM 是如何处理异常的？

在一个方法中如果发生异常，这个方法会创建一个异常对象，并转交给 JVM，该异常对象包含异常名称，异常描述以及异常发生时应用程序的状态。创建异常对象并转交给 JVM 的过程称为抛出异常。可能有一系列的方法调用，最终才进入抛出异常的方法，这一系列方法调用的有序列表叫做调用栈。

JVM 会顺着调用栈去查找看是否有可以处理异常的代码，如果有，则调用异常处理代码。当 JVM 发现可以处理异常的代码时，会把发生的异常传递给它。如果 JVM 没有找到可以处理该异常的代码块，JVM 就会将该异常转交给默认的异常处理器（默认处理器为 JVM 的一部分），默认异常处理器打印出异常信息并终止应用程序。 想要深入了解的小伙伴可以看这篇文章：[https://www.cnblogs.com/qdhxhz/p/10765839.html](https://www.cnblogs.com/qdhxhz/p/10765839.html)


# 6 参考

http://gityuan.com/2016/06/24/app-crash/

https://www.pdai.tech/md/java/basic/java-basic-x-exception.html
