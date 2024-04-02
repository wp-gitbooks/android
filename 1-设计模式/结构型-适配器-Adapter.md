---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 适配器

## 1.1 名称
适配器模式（Adapter）的定义如下：将一个类的接口**转换**成客户希望的另外一个接口，使得原本由于接口不兼容而不能一起工作的那些类能一起工作。适配器模式分为类结构型模式和对象结构型模式两种，前者类之间的耦合度比后者高，且要求程序员了解现有组件库中的相关组件的内部结构，所以应用相对较少些。

Adapter模式的宗旨：**保留现有类所提供的服务，向客户提供接口，以满足客户的期望**。


## 1.2 问题
在现实生活中，经常出现两个对象因接口不兼容而不能在一起工作的实例，这时需要第三者进行适配。
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202205301852143.png)


在软件设计中也可能出现：**需要开发的具有某种业务功能的组件在现有的组件库中已经存在，但它们与当前系统的接口规范不兼容，如果重新开发这些组件成本又很高，这时用适配器模式能很好地解决这些问题。**

## 1.3 解决方案
### 1.3.1 结构-组成
适配器模式（Adapter）包含以下主要角色：
-   目标（Target）接口：当前系统业务所期待的接口，它可以是抽象类或接口。
-   适配者（Adaptee）类：它是被访问和适配的现存组件库中的组件接口。
-   适配器（Adapter）类：它是一个转换器，通过继承或引用适配者的对象，把适配者接口转换成目标接口，让客户按目标接口的格式访问适配者。


### 1.3.2 方式2-类适配器
#### 1.3.2.1 结构
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202210271643720.png)

![Adapter_classModel.jpg](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210401092419.jpg)

#### 1.3.2.2 目标（Target）接口
即图中的欧式三叉
```java
public interface EuropeSocket {
    /** 欧式三叉 通电 接通电 插座*/
    String useEuropesocket();
}

// 欧式三叉实现类
public class EuropeSocketImpl implements EuropeSocket {

    @Override
    public String useEuropesocket() {
        String msg ="使用欧式三叉充电";
        return msg;
    }
}
```
#### 1.3.2.3 适配者（Adaptee）
即中国双叉
```java
public interface ChineseSocket {
    /**
     * 使用中国双叉充电
     * @return
     */
    String useChineseSocket();
}

// 中国插头的实现类
public class ChineseSocketImpl implements ChineseSocket {

    @Override
    public String useChineseSocket() {
        String msg="使用中国双叉充电";
        return msg;
    }
}
```

#### 1.3.2.4 适配器（Adapter）类
```java
/**
 * 定义适配器类 中国双叉转为欧洲三叉
 *
 */
public class ChineseAdapterEurope extends EuropeSocketImpl implements ChineseSocket {

    @Override
    public String useChineseSocket() {
        System.out.println("使用转换器转换完成");
        return useEuropesocket();
    }
}
```

#### 1.3.2.5 测试
电脑类
```java
public class Computer {

    public String useChineseSocket(ChineseSocket chineseSocket) {
        if(chineseSocket == null) {
            throw new NullPointerException("sd card null");
        }
        return chineseSocket.useChineseSocket();
    }
}
```

测试
```java
public class Client {
    public static void main(String[] args) {
        Computer computer = new Computer();
        ChineseSocket chineseSocket = new ChineseSocketImpl();
        System.out.println(computer.useChineseSocket(chineseSocket));
        System.out.println("------------");
        ChineseAdapterEurope adapter = new ChineseAdapterEurope();
        System.out.println(computer.useChineseSocket(adapter));
        /**
         * 输出：
         * 使用中国双叉充电
         * ------------
         * 使用转换器转换完成
         * 使用欧式三叉充电
         */
    }
}
```



### 1.3.3 方式1-对象适配器
#### 1.3.3.1 结构
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202210271643745.png)

![dapter.jpg](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210401092355.jpg)

#### 1.3.3.2 目标（Target）接口
即图中的欧式三叉
```java
public interface EuropeSocket {
    /** 欧式三叉 通电 接通电 插座*/
    String useEuropesocket();
}

// 欧式三叉实现类
public class EuropeSocketImpl implements EuropeSocket {

    @Override
    public String useEuropesocket() {
        String msg ="使用欧式三叉充电";
        return msg;
    }
}
```

#### 1.3.3.3 适配者（Adaptee）
即中国双叉
```java
public interface ChineseSocket {
    /**
     * 使用中国双叉充电
     * @return
     */
    String useChineseSocket();
}

// 中国插头的实现类
public class ChineseSocketImpl implements ChineseSocket {

    @Override
    public String useChineseSocket() {
        String msg="使用中国双叉充电";
        return msg;
    }
}
```

#### 1.3.3.4 适配器（Adapter）类
就是这个适配器内做了一些更改 从继承改为了成员变量的方式
```java
public class ChineseAdapterEurope implements ChineseSocket {

    private EuropeSocket europeSocket;

    public ChineseAdapterEurope(EuropeSocket europeSocket) {
        this.europeSocket = europeSocket;
    }

    @Override
    public String useChineseSocket() {
        System.out.println("使用转换器转换完成");
        return europeSocket.useEuropesocket();
    }
}
```

#### 1.3.3.5 测试
电脑类
```java
public class Computer {

    public String useChineseSocket(ChineseSocket chineseSocket) {
        if(chineseSocket == null) {
            throw new NullPointerException("sd card null");
        }
        return chineseSocket.useChineseSocket();
    }
}
```

测试
```java
public class Client {
    public static void main(String[] args) {
        Computer computer = new Computer();
        ChineseSocket chineseSocket = new ChineseSocketImpl();
        System.out.println(computer.useChineseSocket(chineseSocket));

        System.out.println("------------");
        //这里做了更改
        EuropeSocket europeSocket=new EuropeSocketImpl();
        ChineseAdapterEurope adapter = new ChineseAdapterEurope(europeSocket);
        System.out.println(computer.useChineseSocket(adapter));
        /**
         * 输出：
         * 使用中国双叉充电
         * ------------
         * 使用转换器转换完成
         * 使用欧式三叉充电
         */
    }
}
```

## 1.4 效果
### 1.4.1 应用场景-适用性
适配器模式（Adapter）通常适用于以下场景。
-   以前开发的系统存在满足新系统功能需求的类，但其接口同新系统的接口不一致。（~~就是所谓的加一层，一层不行就加两层~~）😁
-   使用第三方提供的组件，但组件接口定义和自己要求的接口定义不同。
- 系统需要使用现有的类，而这些类的接口不符合系统的需要。
-   想要建立一个可以重复使用的类，用于与一些彼此之间没有太大关联的一些类，包括一些可能在将来引进的类一起工作。
-   需要一个统一的输出接口，而输入端的类型不可预知。


#### 1.4.1.1 适配器模式在Android源码中的应用
[[Android适配器模式-ListView与Adapter]]

### 1.4.2 优缺点
优点：  
* 更好的复用性  
系统需要使用现有的类，而此类的接口不符合系统的需要。那么通过适配器模式就可以让这些功能得到更好的复用。
* 更好的扩展性  
在实现适配器功能的时候，可以调用自己开发的功能，从而自然地扩展系统的功能。

缺点：  
过多的使用适配器，会让系统非常零乱，不易整体进行把握。比如，明明看到调用的是A接口，其实内部被适配成了B接口的实现，一个系统如果太多出现这种情况，无异于一场灾难。因此如果不是很有必要，可以不使用适配器，而是直接对系统进行重构。


