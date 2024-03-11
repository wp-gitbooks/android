# 概述

## 结构

![img-0](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210510095609.png)

# 引用类型

## 强引用Strong Reference

java中的引用默认就是强引用，任何一个对象的赋值操作就产生了对这个对象的强引用。

```
public class StrongReferenceUsage {

    @Test
    public void stringReference(){
        Object obj = new Object();
    }
}
```

强引用的特性是只要有强引用存在，被引用的对象就**不会被垃圾回收**。



## 软引用Soft Reference

软引用在java中有个专门的SoftReference类型，软引用的意思是只有在**内存不足**的情况下，被引用的对象才会被回收。

定义

```
public class SoftReference<T> extends Reference<T>
```

两种构造函数：

```
public SoftReference(T referent)


public SoftReference(T referent, ReferenceQueue<? super T> q)
```



举例：

```
@Test
    public void softReference(){
        Object obj = new Object();
        SoftReference<Object> soft = new SoftReference<>(obj);
        obj = null;
        log.info("{}",soft.get());
        System.gc();
        log.info("{}",soft.get());
    }
```

输出结果：

```
22:50:43.733 [main] INFO com.flydean.SoftReferenceUsage - java.lang.Object@71bc1ae4
22:50:43.749 [main] INFO com.flydean.SoftReferenceUsage - java.lang.Object@71bc1ae4
```



### 使用场景

解决图片缓存问题------->其实效率低下(更好的方式是LruCache算法)



图片缓存。图片缓存框架中，“内存缓存”中的图片是以这种引用保存，使得  JVM 在发生 OOM 之前，可以回收这部分缓存。此外，还可以用在网页缓存上。

```javascript
Browser prev = new Browser();               // 获取页面进行浏览
SoftReference sr = new SoftReference(prev); // 浏览完毕后置为软引用        
if(sr.get()!=null) { 
    rev = (Browser) sr.get();           // 还没有被回收器回收，直接获取
} else {
    prev = new Browser();               // 由于内存吃紧，所以对软引用的对象回收了
    sr = new SoftReference(prev);       // 重新构建
}
```



假如有一个应用需要读取大量的本地图片
每次读取图片都从硬盘读取会影响性能。
一次全部加载到内存中，又可能造成内存溢出。
此时，可以使用软引用解决问题；
使用一个 HashMap 保存图片的路径和响应图片对象关联的软引用之间的映射关系，
内存不足时，jvm 会自动回收这些缓存图片对象所占用的空间，可以避免 OOM。

```java
Map<String,SoftReference<Bigmap>> imageCache=new HashMap<String,SoftReference<Bitmap>>();
```



## 弱引用Weak Reference

weakReference和softReference很类似，不同的是weekReference引用的对象只要**垃圾回收**执行，就会被回收，而不管是否内存不足。



WeakHahsMap 的实现原理简单来说就是**HashMap里面的条目 Entry继承了 WeakReference**，那么当 Entry 的 **key 不再被使用（即，引用对象不可达）且被 GC 后**，那么该 Entry 就会进入到 ReferenceQueue 中。当我们调用WeakHashMap 的get和put方法会有一个副作用，即清除无效key对应的Entry。这个过程就和上面的代码很类似了，首先会从引用队列中取出一个Entry对象，然后在HashMap中查找这个Entry对象的位置，最后把这个 Entry 从 HashMap中删除，这时key和value对象都被回收了。重复这个过程直到队列为空



最后说明一点，**WeakHashMap是线程安全**的。



构造函数：

```
public WeakReference(T referent)；

public WeakReference(T referent, ReferenceQueue<? super T> q)；
```

举例：

```
    @Test
    public void weakReference() throws InterruptedException {
        Object obj = new Object();
        WeakReference<Object> weak = new WeakReference<>(obj);
        obj = null;
        log.info("{}",weak.get());
        System.gc();
        log.info("{}",weak.get());
    }
```

输出结果：

```
22:58:02.019 [main] INFO com.flydean.WeakReferenceUsage - java.lang.Object@71bc1ae4
22:58:02.047 [main] INFO com.flydean.WeakReferenceUsage - null
```

我们看到gc过后，弱引用的对象被回收掉了。



例子2：

```
package javalearning;
 
import java.util.WeakHashMap;
 
public class WeakHashMapDemo {
    public static void main(String[] args){
         
        WeakHashMap<String, byte[]> whm = new WeakHashMap<String, byte[]>();
        String s1 = new String("s1");
        String s2 = new String("s2");
        String s3 = new String("s3");
         
        whm.put(s1, new byte[100]);
        whm.put(s2, new byte[100]);
        whm.put(s3, new byte[100]);
         
        s2 = null;
        s3 = null;
         
        /*此时可能还未执行gc,所以可能还可以通过仅有弱引用的key找到value*/
        System.out.println(whm.get("s1"));
        System.out.println(whm.get("s2"));
        System.out.println(whm.get("s3"));
         
        System.out.println("-------------------");
         
        /*执行gc,导致仅有弱引用的key对应的entry(包括value)全部被回收*/
        System.gc();
        System.out.println(whm.get("s1"));
        System.out.println(whm.get("s2"));
        System.out.println(whm.get("s3"));
    }
}
```



运行结果：

```
[B@2a139a55
[B@15db9742
[B@6d06d69c
-------------------
[B@2a139a55
null
null
```



### 使用场景

#### ThreadLocal

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210510095714.png)

#### WeakHashMap [[WeakHashMap]]

https://blog.csdn.net/crave_shy/article/details/17611009

http://www.javaeye.com/topic/587995


最后讲一下WeakHashMap，WeakHashMap跟WeakReference有点类似，在WeakHashMap如果key不再被使用，被赋值为null的时候，该key对应的Entry会自动从WeakHashMap中删除。

我们举个例子：

```
package javalearning;
 
import java.util.WeakHashMap;
 
public class WeakHashMapDemo {
    public static void main(String[] args){
         
        WeakHashMap<String, byte[]> whm = new WeakHashMap<String, byte[]>();
        String s1 = new String("s1");
        String s2 = new String("s2");
        String s3 = new String("s3");
         
        whm.put(s1, new byte[100]);
        whm.put(s2, new byte[100]);
        whm.put(s3, new byte[100]);
         
        s2 = null;
        s3 = null;
         
        /*此时可能还未执行gc,所以可能还可以通过仅有弱引用的key找到value*/
        System.out.println(whm.get("s1"));
        System.out.println(whm.get("s2"));
        System.out.println(whm.get("s3"));
         
        System.out.println("-------------------");
         
        /*执行gc,导致仅有弱引用的key对应的entry(包括value)全部被回收*/
        System.gc();
        System.out.println(whm.get("s1"));
        System.out.println(whm.get("s2"));
        System.out.println(whm.get("s3"));
    }
}
```

输出结果：

```
[B@2a139a55
[B@15db9742
[B@6d06d69c
-------------------
[B@2a139a55
null
null
```

可以看到gc过后，WeakHashMap只有一个Entry了。





#### 对象引用

在下面的代码中，如果类 B 不是虚引用类 A 的话，执行 main 方法会出现内存泄漏的问题， 因为类 B 依然依赖于 A。

```javascript
public class Main {
    public static void main(String[] args) {

        A a = new A();
        B b = new B(a);
        a = null;
        System.gc();
        System.out.println(b.getA());  // null

    }

}

class A {}

class B {

    WeakReference<A> weakReference;

    public B(A a) {
        weakReference = new WeakReference<>(a);
    }

    public A getA() {
        return weakReference.get();
    }
}
```

在静态内部类中，经常会使用虚引用。例如：一个类发送网络请求，承担 callback 的静态内部类，则常以虚引用的方式来保存外部类的引用，当外部类需要被 JVM 回收时，不会因为网络请求没有及时回应，引起内存泄漏





## 虚引用PhantomReference

PhantomReference的作用是跟踪垃圾回收器收集对象的活动，在GC的过程中，如果发现有PhantomReference，GC则会将引用放到ReferenceQueue中，由程序员自己处理，当程序员调用ReferenceQueue.pull()方法，将引用出ReferenceQueue移除之后，Reference对象会变成Inactive状态，意味着被引用的对象可以被回收了。



和SoftReference和WeakReference不同的是，PhantomReference只有一个构造函数，必须传入ReferenceQueue：



```
public PhantomReference(T referent, ReferenceQueue<? super T> q)
```

例子：

```
@Slf4j
public class PhantomReferenceUsage {

    @Test
    public void usePhantomReference(){
        ReferenceQueue<Object> rq = new ReferenceQueue<>();
        Object obj = new Object();
        PhantomReference<Object> phantomReference = new PhantomReference<>(obj,rq);
        obj = null;
        log.info("{}",phantomReference.get());
        System.gc();
        Reference<Object> r = (Reference<Object>)rq.poll();
        log.info("{}",r);
    }
}
```

输出结果：

```
07:06:46.336 [main] INFO com.flydean.PhantomReferenceUsage - null
07:06:46.353 [main] INFO com.flydean.PhantomReferenceUsage - java.lang.ref.PhantomReference@136432db
```

我们看到get的值是null，而GC过后，poll是有值的。

因为PhantomReference引用的是需要被垃圾回收的对象，所以在类的定义中，get一直都是返回null：

```
    public T get() {
        return null;
    }
```



### 使用场景

#### 虚引用应用-析构资源对象

https://www.zhihu.com/question/49760047/answer/123486092



可以用来跟踪对象呗垃圾回收的活动。一般可以通过虚引用达到回收一些非java内的一些资源比如堆外内存的行为。例如：在 DirectByteBuffer 中，会创建一个 PhantomReference 的子类Cleaner的虚引用实例用来引用该 DirectByteBuffer 实例，Cleaner 创建时会添加一个 Runnable 实例，当被引用的 DirectByteBuffer 对象不可达被垃圾回收时，将会执行 Cleaner 实例内部的 Runnable 实例的 run 方法，用来回收堆外资源。



#### 堆外内存

![![][img-3]](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210510095828.png)





## 回收时机

**强引用  >  软引用  >  弱引用  >  虚引用**



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210510094747.png)



![image-20210510094814286](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210510094837.png)



# ReferenceQueue引用队列

## 是什么

通常我们将其ReferenceQueue翻译为引用队列，换言之就是存放引用的队列，保存的是Reference对象。其作用在于**Reference对象所引用的对象**被GC回收时，该Reference对象将会被加入引用队列中（ReferenceQueue）的队列末尾。

ReferenceQueue常用的方法： 
 `public Reference poll()`：从队列中取出一个元素，队列为空则返回null； 
 `public Reference remove()`：从队列中出对一个元素，若没有则阻塞至有可出队元素； 
 `public Reference remove(long timeout)`：从队列中出对一个元素，若没有则阻塞至有可出对元素或阻塞至超过timeout毫秒；

见如下代码：

```java
ReferenceQueue< Person> rq=new ReferenceQueue();
Person person=new Person();
SoftReference sr=new SoftReference(person,rq);复制代码
```

这段代码中，对于Person对象有两种引用类型，一是person的强引用，而是sr的软引用。sr强引用了SoftReference对象，该对象软引用了Person对象。当person被回收时，sr所强引用的对象将会被放到rq的队列末尾。利用ReferenceQueue可以清除失去了软引用对象的SoftReference,如下操作：

```java
SoftReference ref=null;
while((ref=(Person)rq.poll())!=null){
    
}
```


## 源码解析

https://www.jianshu.com/p/f86d3a43eec5


讲完上面的四种引用,接下来我们谈一下他们的父类Reference和ReferenceQueue的作用。

Reference是一个抽象类，每个Reference都有一个指向的对象，在Reference中有5个非常重要的属性：referent，next，discovered，pending，queue。

```
private T referent;         /* Treated specially by GC */
volatile ReferenceQueue<? super T> queue;
Reference next;
transient private Reference<T> discovered;  /* used by VM */
private static Reference<Object> pending = null;
```

每个Reference都可以看成是一个节点，多个Reference通过next，discovered和pending这三个属性进行关联。

先用一张图来对Reference有个整体的概念：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210414095519.png)

referent就是Reference实际引用的对象。

通过next属性，可以构建ReferenceQueue。

通过discovered属性，可以构建Discovered List。

通过pending属性，可以构建Pending List。

### 四大状态

在讲这三个Queue/List之前，我们先讲一下Reference的四个状态：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210414095536.png)

从上面的图中，我们可以看到一个Reference可以有四个状态。

因为Reference有两个构造函数，一个带ReferenceQueue,一个不带。

```
    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
```

**对于带ReferenceQueue的Reference，GC会把要回收对象的Reference放到ReferenceQueue中，后续该Reference需要程序员自己处理（调用poll方法）**。

**不带ReferenceQueue的Reference，由GC自己处理，待回收的对象其Reference状态会变成Inactive。**

创建好了Reference，就进入active状态。

active状态下，如果引用对象的可到达状态发送变化就会转变成Inactive或Pending状态。

Inactive状态很好理解，到达Inactive状态的Reference状态不能被改变，会等待GC回收。

Pending状态代表等待入Queue，Reference内部有个ReferenceHandler，会调用enqueue方法，将Pending对象入到Queue中。

入Queue的对象，其状态就变成了Enqueued。

Enqueued状态的对象，如果调用poll方法从ReferenceQueue拿出，则该Reference的状态就变成了Inactive，等待GC的回收。

这就是Reference的一个完整的生命周期。

### 三个Queue/List

有了上面四个状态的概念，我们接下来讲三个Queue/List：ReferenceQueue，discovered List和pending List。

ReferenceQueue在讲状态的时候已经讲过了，它本质是由Reference中的next连接而成的。用来存储GC待回收的对象。

pending List就是待入ReferenceQueue的list。

discovered List这个有点特别，在Pending状态时候，discovered List就等于pending List。

在Active状态的时候，discovered List实际上维持的是一个引用链。通过这个引用链，我们可以获得引用的链式结构，当某个Reference状态不再是Active状态时，需要将这个Reference从discovered List中删除。


# 总结

本文讲解了4个java中的引用类型，并深入探讨了Reference的内部机制，感兴趣的小伙伴可以留言一起讨论。


# 参考

https://xie.infoq.cn/article/a01eee563cd3d7e107f961e47

https://www.cnblogs.com/nullzx/p/7406151.html

https://my.oschina.net/flydean/blog/4263694


