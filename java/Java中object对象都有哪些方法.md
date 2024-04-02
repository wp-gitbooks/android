# 概述

**面试官会常问，java的Object类都有那些方法？简单吧，但是很可能不会，今天总结一下。**

先放出JDK1.7中Object类中所有的方法列表如下：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210820173922.jpg)

## 1、toString()

返回一个String类型的字符串，用于描述当前对象的信息，使用频率极高，一般都有子类都有覆盖，即可以重写返回对自己有用的信息，默认返回的是当前对象的类名+hashCode的16进制数字。

```java
public class Test {
    public static class A {
        public String toString(){
            return "this is A";
        }
    }

    public static void main(String[] args) {
        A obj = new A();
        System.out.println(obj);
    }
}
```

**输出：**

```java
this is A
```

------

```java
public class Test {
    public static class A {
        public String getString() {
            return "this is A";
        }
    }
    public static void main(String[] args) {
        A obj = new A();
        System.out.println(obj);
        System.out.println(obj.getString());
    }
}
```

**输出：**

```java
Test$A@1b6d3586       //类名+地址
this is A
```

**toString()作用：**碰到“println（）”之类的输出方法时会自动调用，不用显式打出来。

**扩展：**关于String ,StringBuffer和StringBuilder的区别？

## 2、wait()、wait(long timeout)、wait(long timeout，int naos)

注意在多线程环境下才会使用wait()

> **wait()的作用：**让当前线程进入等待状态，同时，wait()也会让当前线程释放它所持有的锁。“直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法”，当前线程被唤醒并进入“就绪状态”。
> **wait(long timeout)的作用：**让当前线程处于“等待(阻塞)状态”，“直到其他线程调用此对象的notify()方法或 notifyAll() 方法，或者超过指定的时间量”，当前线程被唤醒并进入“就绪状态”。

调用wait()方法后当前线程就进入睡眠状态，直到以下事件发生。

（1）其他线程调用了该对象的notify方法。

（2）其他线程调用了该对象的notifyAll方法。

（3）其他线程调用了interrupt中断该线程。

（4）时间间隔到了。

此时该线程就可以被调度了，如果是被中断的话就抛出一个InterruptedException异常。



**咱们用代码证明一下wait()让当前线程等待**

```java
class ThreadA extends Thread{
    public ThreadA(String name) {
        super(name);
    }
    public void run() {
        synchronized (this) {
            try {
                Thread.sleep(1000); // 使当前线程阻塞1s，确保主程序的 t1.wait(); 执行之后再执行 notify()
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+" call notify()");
            // 唤醒当前的wait线程
            this.notify();
        }
    }
}
public class Test {
    public static void main(String[] args) {
        ThreadA t1 = new ThreadA("t1");
        synchronized(t1) {
            try {
                // 启动“线程t1”
                System.out.println(Thread.currentThread().getName()+" start t1");
                t1.start();
                // 主线程等待t1通过notify()唤醒。
                System.out.println(Thread.currentThread().getName()+" wait()");
                t1.wait();  //  不是使t1线程等待，而是当前执行wait的线程等待
                System.out.println(Thread.currentThread().getName()+" continue");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

**输出：**

```java
main start t1
main wait()
t1 call notify()
main continue
```



**咱们用代码证明一下wait(long timeout)让当前线程超时就被唤醒**

```java
class ThreadA extends Thread{
    public ThreadA(String name) {
        super(name);
    }
    public void run() {
        System.out.println(Thread.currentThread().getName() + " run ");
        // 死循环，不断运行。
        while(true){

        }  //  这个线程与主线程无关，无 synchronized
    }
}
public class Test {
    public static void main(String[] args) {
        ThreadA t1 = new ThreadA("t1");
        synchronized(t1) {
            try {
                // 启动“线程t1”
                System.out.println(Thread.currentThread().getName() + " start t1");
                t1.start();
                // 主线程等待t1通过notify()唤醒 或 notifyAll()唤醒，或超过3000ms延时；然后才被唤醒。
                System.out.println(Thread.currentThread().getName() + " call wait ");
                t1.wait(3000);
                System.out.println(Thread.currentThread().getName() + " continue");
                t1.stop();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

**输出：**

```java
main start t1
main wait()
t1 call notify()
main continue
```

## 3、notify()和notifyAll()

**notify()方法：**针对多线程，在wait()方法后面使用，用于唤醒该对象等待的某个线程

**notifyAll()方法：**针对多线程，在wait()方法后面使用，用于唤醒该对象等待的所有线程

## 4、finalize()

java允许在类中定义一个名为finalize()的方法。

它的工作原理是：在垃圾回收器准备好释放对象占用的存储空间前，将首先调用其finalize()方法，并且在**下一次**垃圾回收动作发生时，才会真正回收对象占用的内存。

## 5、equals()

Object中的equals()方法是直接用来判断两个对象指向的内存空间是不是同一块。如果是同一块内存地址，则返回true。

## 6、hashCode()

```java
public class Test {
    public static void main(String args[]) {
        String str = new String("https://zhuanlan.zhihu.com/p/93556076/edit");
        System.out.println("字符串的哈希码为 :" + str.hashCode() );
    }
}
```

**输出：**

```java
字符串的哈希码为 :-1951809095
```

作用：返回字符串的哈希码值，用于判断两个对象的地址是不是同一个。注意与上面equals()方法的区别哈。

## 7、getClass()

返回Object运行时的类，不可重写，要调用的话，一般和getName()联合使用，如getClass().getName();

```java
class Person{
    int id;
    String name;
    public Person(int id, String name) {
        super();
        this.id = id;
        this.name = name;
    }
}

public class Test {
    public static void main(String[] args) {
        Person p = new Person(1,"小明");
        System.out.println(p.getClass());
        System.out.println(p.getClass().getName());
    }
}
```

**输出：**

```java
class Person
Person
```

注意：getClass(）返回的是内存中实实在在存在的Person 这个类

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210820173922.jpg)

## 8、clone()

实际中都用new，知道clone()方法是干嘛的就行。



# 参考

https://zhuanlan.zhihu.com/p/93556076