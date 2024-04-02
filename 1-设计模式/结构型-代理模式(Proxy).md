---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 代理模式/委托模式
[[结构型-代理模式-rich]]
## 1.1 名称
代理模式/委托模式

代理模式的定义：为其他对象提供一种[代理](https://baike.baidu.com/item/%E4%BB%A3%E7%90%86?fromModule=lemma_inlink)以**控制对这个对象的访问**。在某些情况下，一个对象不适合或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用。

1. **定义：给目标对象提供一个代理对象，并由代理对象控制目标对象的引用**
2. **目的：通过引入代理的方式来间接访问目标对象，防止直接访问目标对象给系统带来不确定的复杂性**

> 为其他对象提供一种代理以控制对这个对象的访问。

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202210281122008.png)

### 1.1.1 类比-例子
我们可以理解为生活中常见的中介或者明星经纪人，我们买房一般都会通过中介，但是最后卖房的却是开发商，可以认为中介就是开发商的代理。



## 1.2 问题
主要解决的问题是：在**直接访问对象**时带来的问题。（某些对象只想提供部分方法让外部对象访问）

为什么要控制对于某个对象的访问呢？

> 当不能访问或不想直接访问或访问某个对象存在困难时，我们可以通过一个代理对象来间接访问，为了客户端使用的透明性，我们应该保证代理对象和被代理对象应该实现同一个接口。

## 1.3 解决方案
### 1.3.1 静态代理
#### 1.3.1.1 结构
![Proxy](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210401093138.jpg)
![image-20210718111906998](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210718111907.png)

#### 1.3.1.2 静态代理使用
##### 1.3.1.2.1 静态代理需实现的方法

```java
public interface Subject {

    void sayGoodBye();

    void sayHello(String str);

    boolean isProxy();
}

```

##### 1.3.1.2.2 定义被代理类（原来功能类）并实现被代理类的功能逻辑（打死都不改的那种）

```java
public class RealSubject implements Subject {
    @Override
    public void sayGoodBye() {
        System.out.println("RealSubject 我是原封不动的代码 sayGoodBye ");
    }
    @Override
    public void sayHello(String str) {
        System.out.println("RealSubject 我是原封不动的代码 sayHello  " + str);
    }

    @Override
    public boolean isProxy() {
        System.out.println("RealSubject  我是原封不动的代码    isProxy     ");
        return false;
    }
}

```

##### 1.3.1.2.3 定义静态代理类（功能增加类），这个代理类也必须要实现和被代理类相同的Subject接口，便于对原有功能的增强

```java
public class ProxySubject implements Subject {
    private Subject subject;

    public ProxySubject(Subject subject) {
        this.subject = subject;
    }

    @Override
    public void sayGoodBye() {
        //代理类，功能的增强 调用前 sayGoodBye 可做操作（比如是否能调用的权限认证）
        System.out.println("ProxySubject sayGoodBye begin  " +
                "代理类，功能的增强 调用前 sayGoodBye 可做操作（比如是否能调用的权限认证）");
        //在代理类的方法中 间接访问被代理对象的方法
        subject.sayGoodBye();
        System.out.println("ProxySubject sayGoodBye end " +
                "这里可处理原方法调用后的逻辑处理");
    }

    @Override
    public void sayHello(String str) {
        //代理类，功能的增强 调用前 sayHello 可做操作（比如是否能调用的权限认证） 并测试带参数的方法
        System.out.println("ProxySubject sayHello begin  " +
                "代理类，功能的增强 调用前 sayHello 可做操作（比如是否能调用的权限认证）");
        //在代理类的方法中 间接访问被代理对象的方法
        subject.sayHello(str);
        System.out.println("ProxySubject sayHello end " +
                "这里可处理原方法调用后的逻辑处理");
    }

    @Override
    public boolean isProxy() {
        //代理类，功能的增强 调用前 isProxy 可做操作（比如是否能调用的权限认证） 并测试带返回的方法
        System.out.println("ProxySubject isProxy begin  " +
                "代理类，功能的增强 调用前 isProxy 可做操作（比如是否能调用的权限认证）");
        boolean boolReturn = subject.isProxy();
        System.out.println("ProxySubject isProxy end " +
                "这里可处理原方法调用后的逻辑处理");
        return boolReturn;
    }
}
```

##### 1.3.1.2.4 静态代理使用

```java
public class ProxyMain {

    public static void main(String[] args) {
        //被代理的对象，某些情况下 我们不希望修改已有的代码，我们采用代理来间接访问
        RealSubject realSubject = new RealSubject();
        //代理类对象，将原有代码不想修改的对象传入代理类对象
        ProxySubject proxySubject = new ProxySubject(realSubject);
        //调用代理类对象的方法
        proxySubject.sayGoodBye();
        System.out.println("******");
        proxySubject.sayHello("Test");
        System.out.println("******");
        proxySubject.isProxy();
    }

}
```

##### 1.3.1.2.5 最终打印

```java
ProxySubject sayGoodBye begin  代理类，功能的增强 调用前 sayGoodBye 可做操作（比如是否能调用的权限认证）

RealSubject 我是原封不动的代码 sayGoodBye 

ProxySubject sayGoodBye end 这里可处理原方法调用后的逻辑处理

******

ProxySubject sayHello begin  代理类，功能的增强 调用前 sayHello 可做操作（比如是否能调用的权限认证）

RealSubject 我是原封不动的代码 sayHello  Test

ProxySubject sayHello end 这里可处理原方法调用后的逻辑处理

******

ProxySubject isProxy begin  代理类，功能的增强 调用前 isProxy 可做操作（比如是否能调用的权限认证）

RealSubject  我是原封不动的代码    isProxy  

ProxySubject isProxy end 这里可处理原方法调用后的逻辑处理   
```

##### 1.3.1.2.6 总结
静态代理（传统代理模）的实现方式比较暴力直接，需要将所有被代理类的**所有方法**都写一遍，并且一个个的手动转发过去，麻烦并繁琐。所以我们要学习并使用动态代理。


#### 1.3.1.3 优缺点
优点：
1、在符合开闭原则[[0-设计模式七大原则#5 开闭原则（重点）]]的情况下，对目标对象功能进行扩展和拦截。
2、在访问无法访问的资源，增强现有的接口业务功能方面有很大的优点。
3、一个代理类只能代理一个真实的对象。

缺点：
1、需要为**每个目标类创建代理类和接口，导致类的数量大大增加**，工作量大，导致系统结构比较臃肿和松散。
2、接口功能一旦修改，代理类和目标类也得相应修改，不易维护。


### 1.3.2 动态代理
![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210718111925)

#### 1.3.2.1 什么是动态代理？
动态代理利用了[JDK API](https://link.segmentfault.com/?url=http%3A%2F%2Ftool.oschina.net%2Fuploads%2Fapidocs%2Fjdk-zh%2F)，动态地在内存中构建代理对象，从而实现对目标对象的代理功能。动态代理又被称为JDK代理或接口代理。

#### 1.3.2.2 为什么要使用动态代理？
因为一个静态代理类只能服务一种类型的目标对象，在目标对象较多的情况下，**会出现代理类较多、代码量较大的问题**。而使用动态代理动态生成代理者对象能避免这种情况的发生。

#### 1.3.2.3 动态代理原理
[[动态代理]]

#### 1.3.2.4 动态代理使用
##### 1.3.2.4.1 在java的动态代理机制中，有两个重要的类或接口
- **一个是 InvocationHandler(Interface) 需要代码里需要实现该接口**
- **一个则是Proxy(Class)**

##### 1.3.2.4.2 InvocationHandler (Interface)  详解

```java
public interface InvocationHandler {

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
复制代码
```

- proxy：指代生成的代理对象；
- method：指代的是我们所要调用真实对象的某个方法的Method对象；
- args：指代的是调用真实对象某个方法时接受的参数；
- 返回值 Object ：指的是需要原封不动的返回，被代理所调用的方法的返回

##### 1.3.2.4.3 Proxy (Class) 核心原理
- 编译时，代理对象的class并不存在，当需要调用 **Proxy.newProxyInstance** 方法时，会构建一个Proxy0的class字节码，并且加载到内存

##### 1.3.2.4.4 Proxy.newProxyInstance方法详解

```java
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {....}
复制代码
```

- loader：一个ClassLoader对象，定义了由哪个ClassLoader对象来对生成的代理对象进行加载
- interfaces：一个Interface对象的数组，表示的是我将要给我需要代理的对象提供一组什么接口，如果我提供了一组接口给它，那么这个代理对象就宣称实现了该接口(多态)，这样我就能调用这组接口中的方法了
- 一个InvocationHandler对象，表示的是当我这个动态代理对象在调用方法的时候，会关联到哪一个InvocationHandler对象上。

#### 1.3.2.5 动态代理使用例子
##### 1.3.2.5.1 第一步：定义一个接口 （待实现的逻辑）
```java
public interface Subject {

    void sayGoodBye();

    void sayHello(String str);

    boolean isProxy();
}

```

##### 1.3.2.5.2 第二步：定义真实对象（被代理类，打死都不改的那种）：

```java
public class RealSubject implements Subject {
    @Override
    public void sayGoodBye() {
        System.out.println("RealSubject 我是原封不动的代码 sayGoodBye ");
    }
    @Override
    public void sayHello(String str) {
        System.out.println("RealSubject 我是原封不动的代码 sayHello  " + str);
    }

    @Override
    public boolean isProxy() {
        System.out.println("RealSubject  我是原封不动的代码    isProxy     ");
        return false;
    }
}
```

##### 1.3.2.5.3 第三步：定义一个InvocationHandler， 相当于一个代理处理器

```java
public class SubjectInvocationHandler implements InvocationHandler {
    //这个就是我们要代理的真实对象
    private Object subject;

    //构造方法，给我们要代理的真实对象赋初值
    public SubjectInvocationHandler(Object subject) {
        this.subject = subject;
    }

    @Override
    public Object invoke(Object object, Method method, Object[] args) throws Throwable {
        //在代理真实对象前我们可以添加一些自己的操作
        System.out.println("before Method invoke  代理类，功能的增强 调用前 "+method+" 可做操作（比如是否能调用的权限认证）");
        System.out.println("Method:" + method);
        //当代理对象调用真实对象的方法时，其会自动的跳转到代理对象关联的handler对象的invoke方法来进行调用，得到对应的返回值，最后将对应返回值返回
        Object obj = method.invoke(subject, args);
        //在代理真实对象后我们也可以添加一些自己的操作
        System.out.println("after Method invoke  这里可处理原方法调用后的逻辑处理 ");
        return obj;
    }
}
```

##### 1.3.2.5.4 第四步：调用
```java
public class ProxyMain {

    public static void main(String[] args) {


        //被代理类
        Subject realSubject = new RealSubject();
        //我们要代理哪个类，就将该对象传进去，最后是通过该被代理对象来调用其方法的
        SubjectInvocationHandler handler = new SubjectInvocationHandler(realSubject);
        //生成代理类
        Subject subject = (Subject) Proxy.newProxyInstance(handler.getClass().getClassLoader(),
                realSubject.getClass().getInterfaces(), handler);
        //输出代理类对象
        System.out.println("Proxy : " + subject.getClass().getName());
        System.out.println("Proxy super : " + subject.getClass().getSuperclass().getName());
        System.out.println("Proxy interfaces : " + subject.getClass().getInterfaces()[0].getName());
        //调用代理类sayGoodBye方法
        subject.sayGoodBye();
        System.out.println("--------");
        //调用代理类sayHello方法
        subject.sayHello("Test");
        System.out.println("--------");
        System.out.println("---subject.isProxy()-----" + subject.isProxy());
    }

}
```

##### 1.3.2.5.5 最终打印：
```java
Proxy : com.sun.proxy.$Proxy0
Proxy super : java.lang.reflect.Proxy
Proxy interfaces : staticproxy.Subject
before Method invoke  代理类，功能的增强 调用前 public abstract void staticproxy.Subject.sayGoodBye() 可做操作（比如是否能调用的权限认证）
Method:public abstract void staticproxy.Subject.sayGoodBye()
RealSubject 我是原封不动的代码 sayGoodBye 
after Method invoke  这里可处理原方法调用后的逻辑处理 
--------
before Method invoke  代理类，功能的增强 调用前 public abstract void staticproxy.Subject.sayHello(java.lang.String) 可做操作（比如是否能调用的权限认证）
Method:public abstract void staticproxy.Subject.sayHello(java.lang.String)
RealSubject 我是原封不动的代码 sayHello  Test
after Method invoke  这里可处理原方法调用后的逻辑处理 
--------
before Method invoke  代理类，功能的增强 调用前 public abstract boolean staticproxy.Subject.isProxy() 可做操作（比如是否能调用的权限认证）
Method:public abstract boolean staticproxy.Subject.isProxy()
RealSubject  我是原封不动的代码    isProxy     
after Method invoke  这里可处理原方法调用后的逻辑处理 
---subject.isProxy()-----false
---subject.isProxy()-----false
复制代码
```

#### 1.3.2.6 动态代理总结
- 动态代理能够增加程序的灵活度，比如调用方法前后的逻辑处理
- 完美解决解耦问题，动态代理可以将**调用层和实现层**分离
- 动态代理不需要接口实现类
- 动态代理可以解决程序执行流程（下期讲解事件转到activity执行）


### 1.3.3 静态代理和动态代理
[[静态代理和动态代理]]
根据加载被代理类的时机不同，将代理分为静态代理和动态代理。
- **静态代理：编译时就确定了被代理的类是哪一个**
- **动态代理：运行时才确定被代理的类是哪个**
动态代理只能对**接口**使用，无法对普通类和抽象类使用。


#### 1.3.3.1 如何区分静态代理和动态代理？
- 静态代理：程序运行前，代理类已经存在。
- 动态代理：程序运行前，代理类不存在，运行过程中，动态生成代理类。

代理实现方式：如果按照**代理创建的时期**来进行分类的话， 可以分为静态代理、动态代理。

一：静态代理是由程序员创建或特定工具自动生成代理类，再对其编译，在程序运行之前，代理类.class文件就已经被创建了。

二：动态代理是在程序运行时通过反射机制动态创建代理对象。

图解：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210718112736)



图解二：
![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210718112751)

类加载详细流程：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210718112802)






## 1.4 效果
### 1.4.1 应用场景
#### 1.4.1.1 远程代理(AIDL)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210720200458.png)


#### 1.4.1.2 保护代理(权限控制)
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210720200540.png)



#### 1.4.1.3 虚代理(图片占位)
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210720201131.png)



#### 1.4.1.4 协作开发(Mock Class)
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210720201234.png)


#### 1.4.1.5 AOP-给生活加点料(记日志)
[[字节码增强]]
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210720201346.png)








## 1.5 参考
<https://juejin.im/post/6844903978342301709>

[https://blog.csdn.net/mine\_song/article/details/71373305](https://blog.csdn.net/mine_song/article/details/71373305)

<https://segmentfault.com/a/1190000011291179>


