---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 享元模式

## 1.1 名称
使用共享对象有效的支持大量的细粒度对象的复用。

**对象状态：内部不变、外部变**

### 1.1.1 是什么
**享元模式也是为了减少内存的使用，避免出现大量重复的创建销毁对象的场景**。

**享元模式**用在一批相同或相似的对象上，这些对象有可以**共享的内部状态**和**各自不同的外部状态**。

享元模式中会有一个工厂，工厂维护着一个容器，容器以键值对的方式存储，键是对象的内部状态，也就是共享的部分，值就是对象本身。

## 1.2 问题
在有**大量对象**时，有可能会造成内存溢出，我们把其中共同的部分抽象出来，如果有相同的业务请求，直接返回在内存中已有的对象，避免重新创建。

#设计模式/扩展性
面向对象技术可以很好地解决一些灵活性或可**扩展性**问题，但在很多情况下需要在系统中增加类和对象的个数。当对象数量太多时，将导致运行代价过高，带来性能下降等问题。享元模式正是为解决这一类问题而诞生的。享元模式通过共享技术实现相同或相似对象的重用，示意图如下(我们可以共用一个 Hello world 对象，其中字符串 “Hello world” 为内部状态，可共享；字体颜色为外部状态，不可共享，由客户端设定)：

![image-20210624143226567](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210624143226.png)

在享元模式中可以共享的相同内容称为 **内部状态**(Intrinsic State)，而那些需要外部环境来设置的不能共享的内容称为 **外部状态**(Extrinsic State)，其中外部状态和内部状态是相互独立的，外部状态的变化不会引起内部状态的变化。由于区分了内部状态和外部状态，因此可以通过设置不同的外部状态使得相同的对象可以具有一些不同的特征，而相同的内部状态是可以共享的。也就是说，享元模式的本质是分离与共享 ： 分离变与不变，并且共享不变。把一个对象的状态分成内部状态和外部状态，内部状态即是不变的，外部状态是变化的；然后通过共享不变的部分，达到减少对象数量并节约内存的目的。

在享元模式中通常会出现工厂模式，需要创建一个**享元工厂来负责维护一个享元池**(Flyweight Pool)（用于存储具有相同内部状态的享元对象）。在享元模式中，共享的是享元对象的内部状态，外部状态需要通过环境来设置。在实际使用中，能够共享的内部状态是有限的，因此享元对象一般都设计为较小的对象，它所包含的内部状态较少，这种对象也称为 细粒度对象。  享元模式的目的就是使用共享技术来实现大量细粒度对象的复用。

## 1.3 解决方案
### 1.3.1 单纯享元



### 1.3.2 复合享元

#### 1.3.2.1 组成-解决方案

![这里写图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210624142042.png)

- Flyweight:享元对象抽象类或接口。
- ConcreteFlyweight：具体的享元对象
- FlyweightFactory：享元工厂，管理对象池和创建享元对象

##### 1.3.2.1.1 关系
1 对多，一个享元工厂对应多个具体的享元对象

##### 1.3.2.1.2 例子

###### 1.3.2.1.2.1 抽象的车票
```
public interface Ticket {
    void showInfo(String type);
}
```


###### 1.3.2.1.2.2 具体的车票
```
public class ConcreteTicket implements Ticket {
    private String from;
    private String to;
    private int price;
    private String type;

    public ConcreteTicket(String from, String to) {
        this.from = from;
        this.to = to;

    }

    @Override
    public void showInfo(String type) {
        price = new Random().nextInt(500);
        this.type=type;
        System.out.println("从"+from+"到"+to+"的"+this.type+"票价是"+price);
    }
}
```


###### 1.3.2.1.2.3 车票工厂
```
public class TicketFactory {
    private static Map<String,Ticket> tickets = new HashMap<>();

    public static Ticket getTicket(String from,String to){
        String key = from+to;
        if (tickets.containsKey(key)){
            System.out.println("从缓存中获取");
            return tickets.get(key);
        }else {
            System.out.println("新建对象");
            Ticket ticket = new ConcreteTicket(from,to);
            tickets.put(key,ticket);
            return ticket;
        }
    }
}
```


###### 1.3.2.1.2.4 客户端调用
```
public class Client {
    public static void main(String[] args) {
        TicketFactory.getTicket("A","B").showInfo("硬座");
        TicketFactory.getTicket("A","B").showInfo("硬卧");
        TicketFactory.getTicket("C","B").showInfo("硬卧");
    }
}
```

输出：
![这里写图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210624142225.png)




## 1.4 效果
### 1.4.1 应用场景
[[创建型-原型模式]]
- 系统存在**大量相似或相同的对象**。
- 这些对象有较接近的外部状态。
- 需要缓冲池时

#### 1.4.1.1 java
线程池、缓存、连接池、字符串常量池
https://juejin.cn/post/6844903919995322382


##### 1.4.1.1.1 String
```
        String s0 = "abc";
        String s1 = "abc";

        System.out.println("s0 == s1 " + s0 == s1);
```
输出结果：
```java
s0 == s1 true
```

可以看到`s0`和`s1`指向了同一个引用。
由于`String`采用了享元模式，可以防止程序创建过多相同的字符串，节省了内存。

#### 1.4.1.2 Android-Message.obtainMessage
[[0-Handler#Message]]
我们进去看Handler的源码，obtainMessage如下

```java
    public final Message obtainMessage(int what, int arg1, int arg2, Object obj)
    {
        return Message.obtain(this, what, arg1, arg2, obj);
    }
```

我们发现，其中调用了Message.obtain，我们再去看Message的实现

```java
    public static Message obtain(Handler h, int what, 
            int arg1, int arg2, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        m.obj = obj;

        return m;
    }
    
    //获取一个空message
    public static Message obtain() {
        synchronized (sPoolSync) {
            //从对象Message中获取message
            if (sPool != null) {   //private static Message sPool;
                Message m = sPool;
                sPool = m.next;
                //清空message属性
                m.next = null;
                m.flags = 0; // clear in-use flag 
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

我们可以看到，sPool指向的是一个链表，其实这个链表是存储我们用过的Message的，当我们obtain一个Message的时候，会去取链表中的第一个，并把sPool指向下一个，同时把取到的置空。

```java
    /*Message的回收方法*/
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

    /*将用过的Message放入链表中*/
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

我们可以知道**Message的享元模式不是使用的传统的map方式，而是自己构建一个链表**，灵活使用我们的享元模式思想才是重点。

#### 1.4.1.3 线程池中线程


### 1.4.2 优缺点

#### 1.4.2.1 优点
- 大大减少系统创建的对象，降低内存总对象的数量，降低程序占用的内存，增强系统的性能。

#### 1.4.2.2 缺点
- 将对象分为内部状态和外部状态两部分，导致**系统变复杂**，逻辑也更复杂。
- 将享元对象的状态外部化，而读取外部状态使得运行时间稍微变长。

#### 1.4.2.3 总结
享元模式的核心就在**享元工厂**，因为享元对象有可共享的**内部状态**部分和不可共享的**外部状态**部分，因此，内部可共享的就交给工厂去维护处理了，而外部可变的就可以交给客户端去实现。


## 1.5 享元模式与其他模式的关联
在享元模式的享元工厂类中通常提供一个静态的工厂方法用于返回享元对象，使用简单工厂模式来生成享元对象；

在一个系统中，通常只有唯一一个享元工厂，因此享元工厂类可以使用单例模式进行设计；

享元模式可以结合组合模式形成复合享元模式，统一对享元对象设置外部状态


