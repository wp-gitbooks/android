## 1. 引言

> 在计算机程序设计中，**回调函数**，简称**回调**（Callback），是指通过函数参数传递到其他代码的，某一块可执行代码的引用。这一设计允许了底层代码调用在高层定义的子程序。

以上是维基百科对“回调函数”的定义。对于回调，不同的语言有不同的回调形式，例如：

-   C、C++ 允许将函数指针作为参数传递；
-   JavaScript、Python 允许将函数名作为参数传递。

本文将介绍 Java 实现回调的四种写法：

-   反射；
-   直接调用；
-   接口调用；
-   Lambda表达式。

在开始之前，先介绍下本文代码示例的背景，在 main 函数中，我们异步发送一个请求，并且指定处理响应的回调函数，接着 main 函数去做其他事，而当响应到达后，执行回调函数。

## 2. 反射

Java 的反射机制允许我们获取类的信息，其中包括类的方法。我们将以 Method 类型去获取回调函数，然后传递给请求函数。示例如下：

Request 类中的 send 方法有两个参数 clazz、method，分别是Class 类型和 Method 类型，这里的 method 参数就是待传入的回调函数，而为了通过 invoke 方法进行反射调用，还需要一个实例，所以将回调函数所在的类的 Class 对象作为参数传递进来，通过 newInstance 构造一个对象，将顺利通过 invoke 反射调用。

```
public class Request{
    public void send(Class clazz, Method method) throws Exception {
        // 模拟等待响应
        Thread.sleep(3000);
        System.out.println("[Request]:收到响应");
        method.invoke(clazz.newInstance());
    }
}
复制代码
```

CallBack 类很简单，只有一个 processResponse 方法，用于当作回调函数，处理响应。

```
public class CallBack {
    public void processResponse() {
        System.out.println("[CallBack]:处理响应");
    }
}
复制代码
```

我们在 main 方法中，新开了一个线程去发送请求，并且把需要的 CallBack.class 和 processResponse 方法传递进去。

```
public class Main {
    public static void main(String[] args) throws Exception {
        Request request = new Request();
        System.out.println("[Main]:我开个线程去异步发请求");
        new Thread(() -> {
            try {
                request.send(CallBack.class, CallBack.class.getMethod("processResponse"));
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
        System.out.println("[Main]:请求发完了，我去干点别的");
        Thread.sleep(100000);
    }
}
/** Output:
[Main]:我开个线程去异步发请求
[Main]:请求发完了，我去干点别的
[Request]:收到响应
[CallBack]:处理响应
*/
复制代码
```

这种写法需要传递的参数十分繁琐。下面介绍一种简单的写法，直接调用。

## 3. 直接调用

我们来改写下 send 方法的参数，改为一个 CallBack 类型参数。如下：

在 send 方法中我们不使用反射，改为直接通过对象来调用方法。

```
public class Request{
    public void send(CallBack callBack) throws Exception {
        // 模拟等待响应
        Thread.sleep(3000);
        System.out.println("[Request]:收到响应");
        callBack.processResponse();
    }
}
复制代码
```

main 函数中，我们 new 了一个 CallBack 对象作为参数传递给 send 方法。

```
public class Main {
    public static void main(String[] args) throws Exception {
        Request request = new Request();
        System.out.println("[Main]:我开个线程去异步发请求");
        CallBack callBack = new CallBack();
        new Thread(() -> {
            try {
                request.send(callBack);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
        System.out.println("[Main]:请求发完了，我去干点别的");
        Thread.sleep(100000);
    }
}
复制代码
```

这种实现方式十分简单，但是存在的问题是不符合修改封闭原则。也就是说当我们想要换一种“处理响应”的方法时，将必须去修改 CallBack 类的 processRequest()方法。而如果将 CallBack 类改为接口，我们就可以仅更换 CallBack 的实现了。下面请看接口调用的写法。

## 4. 接口调用

首先将 CallBack 类改为接口。

```
public interface CallBack {
    public void processResponse();
}
复制代码
```

再新增一个 CallBack 接口的实现类 CallBackImpl。

```
public class CallBackImpl implements CallBack {
    @Override
    public void processResponse() {
        System.out.println("[CallBack]:处理响应");
    }
}
复制代码
```

Request 类不变。Main 类中的 main 方法将实例化一个 CallBackImpl，然后通过 CallBack 接口传递进去。

```
public class Main {
    public static void main(String[] args) throws Exception {
        Request request = new Request();
        System.out.println("[Main]:我开个线程去异步发请求");
        CallBack callBack = new CallBackImpl();
        new Thread(() -> {
            try {
                request.send(callBack);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
        System.out.println("[Main]:请求发完了，我去干点别的");
        Thread.sleep(100000);
    }
}
复制代码
```

## 5. Lambda表达式

上述方法已经介绍的差不多了，最后我们再介绍一种更加简洁的写法，通过使用 Lamda 表达式，将不用新增一个 CallBack 接口的实现类。下面请看改写的 main 方法：

```
public class Main {
    public static void main(String[] args) throws Exception {
        Request request = new Request();
        System.out.println("[Main]:我开个线程去异步发请求");
        new Thread(() -> {
            try {
                request.send(()-> System.out.println("[CallBack]:处理响应"));
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
        System.out.println("[Main]:请求发完了，我去干点别的");
        Thread.sleep(100000);
    }
}
复制代码
```

我们既不用去新增实现类，也不用去实例化，只需要传递 Lambda 表达式就可以完成回调了。

## 6. 总结

为了让大家更好的理解回调，本文一共介绍了 4 种写法，大家都可以根据自己的需要自取。

# 同步回调和异步回调
（1）同步调用

![](https://images2015.cnblogs.com/blog/801753/201702/801753-20170221201001413-1766758208.png)

同步调用是最基本并且最简单的一种调用方式，类A的方法a()调用类B的方法b()，一直等待b()方法执行完毕，a()方法继续往下走。这种调用方式适用于方法b()执行时间不长的情况，因为b()方法执行时间一长或者直接阻塞的话，a()方法的余下代码是无法执行下去的，这样会造成整个流程的阻塞。

（2）异步调用

![](https://images2015.cnblogs.com/blog/801753/201702/801753-20170221201512429-1532730453.png)

异步调用是为了解决同步调用可能出现阻塞，导致整个流程卡住而产生的一种调用方式。类A的方法方法a()通过新起线程的方式调用类B的方法b()，代码接着直接往下执行，这样无论方法b()执行时间多久，都不会阻塞住方法a()的执行。但是这种方式，由于方法a()不等待方法b()的执行完成，在方法a()需要方法b()执行结果的情况下（视具体业务而定，有些业务比如启异步线程发个微信通知、刷新一个缓存这种就没必要），必须通过一定的方式对方法b()的执行结果进行监听。在Java中，可以使用Future+Callable的方式做到这一点，具体做法可以参见我的这篇文章[Java多线程21：多线程下其他组件之CyclicBarrier、Callable、Future和FutureTask](http://www.cnblogs.com/xrq730/p/4872722.html)。