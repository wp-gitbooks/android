# 概述

## 什么是ThreadLocal？

通过**ThreadLocal**可以解决`多线程读`共享数据的问题，因为共享数据会被复制到每个线程，不需要加锁便可同步访问。但**ThreadLocal**解决不了`多线程写`共享数据的问题，因为每个线程写的都是自己本线程的局部变量，并没将写数据的结果同步到其他线程。理解了这一点，才能理解所谓的：

- **ThreadLocal**以空间换时间，提升多线程并发的效率。什么意思呢？每个线程都有一个**ThreadLocalMap**映射表，正是利用了这个映射表所占用的空间，使得多个线程都可以访问自己的这片空间，不用担心考虑线程同步问题，效率自然会高。
- **ThreadLocal**并不是为了解决共享数据的**互斥写**问题，而是通过一种编程手段，正好提供了**并行读**的功能。什么意思呢？**ThreadLocal**并不是万能的，它的设计初衷只是提供一个便利性，使得线程可以更为方便地使用局部变量。
- **ThreadLocal**提供了一种线程全域访问功能，什么意思呢？一旦将一个对象添加到**ThreadLocal**中，只要不移除它，那么，在线程的生命周期内的任何地方，都可以通过**ThreadLocal.get()**方法拿到这个对象。有时候，代码逻辑比较复杂，一个线程的代码可能分散在很多地方，利用**ThreadLocal**这种便利性，就能简化编程逻辑。



## ThreadLocal出现的背景是什么？

但存在这样一个场景：还是以电商系统为例。买家在访问订单详情页的时候，在不同的条件下会查订单（查数据库）。查库涉及io，对系统的开销和响应时间有较大的影响。由于订单详情页的渲染都是一些读操作，没有写操作，所以，需要在查数据库时做一层本地缓存。而且这个本地缓存是对线程敏感的，只在当前线程生效，别的线程无法访问这个缓存，也就是说线程间是隔离的。

上面只是以电商为例， 所以需要一种方式，能够实现变量的线程间隔离，此变量只能在当前线程生效，不同的线程变量有不同的值。基于以上诉求，java诞生了ThreadLocal，**主要是为了解决内存的线程隔离**。





## 解决了什么问题？

**主要是为了解决内存的线程隔离**。









# 使用

## ThreadLocal的使用方法是什么？

## 使用的效果如何？



# 原理-ThreadLocal是如何实现它的功能的，即ThreadLocal的原理是什么？

线程间的变量相互隔离，我们会怎样设计？
每一个线程，其执行均是依靠Thread类的实例的start方法来启动线程，然后CPU来执行线程。每一个Thread类的实例的运行即为一个线程。若要**每个线程（每个Thread实例）的变量空间隔离，则需要将这个变量的定义声明在Thread这个类中**。这样，每个实例都有属于自己的这个变量的空间，则实现了线程的隔离。事实上，ThreadLocal的源码也是这样实现的。

## 数据结构

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801224126.png)

每个线程对应一个Thread对象，Thread对象中，有一个ThreadLocal.ThreadLocalMap成员变量。
ThreadLocalMap 类似于HashMap，维护的都是key-value键值对，不同的是，HashMap数据结构是数组+链表/红黑树，而ThreadLocalMap数据结构为数组。
ThreadLocalMap 数组中存放的是静态内部类对象Entry(ThreadLocal<?> k, Object v)，可以简单的认为，ThreadLocal 对象为key，set的内容为value。（ 实际上key为弱引用WeakReference<ThreadLocal<?>> ）









## 1 实现内存线程间隔离的原理

在Thread类中声明一个公共的类变量ThreadLocalMap，用以在Thread的实例中预占空间

```
ThreadLocal.ThreadLocalMap threadLocals = null;
```

在ThreadLocal中创建一个内部类ThreadLocalMap，这个Map的key是ThreadLoca对象，value是set进去的ThreadLocal中泛型类型的值

```
private void set(ThreadLocal key, Object value) {...}
```

在new ThreadLocal时，只是简单的创建了个ThreadLocal对象，与线程还没有任何关系，真正产生关系的是在向ThreadLocal对象中set值得时候：
1.首先从当前的线程中获取ThreadLocalMap，如果为空，则初始化当前线程的ThreadLocalMap
2.然后将值set到这个Map中去，如果不为空，则说明当前线程之前已经set过ThreadLocal对象了。
这样用一个ThreadHashMap来存储当前线程的若干个可以线程间隔离的变量，key是ThreadLocal对象，value是要存储的值（类型是ThreadLocal的泛型）

```
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

从ThreadLocal中获取值 ：还是先从当前线程中获取ThreadLocalMap,然后使用ThreadLocal对象(key)去获取这个对象对应的值(value)

```
 public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

到这里，如果仅仅是理解ThreadLocal是如何实现的线程级别的隔离已经完全足够了。简单的讲，就是在Thread的类中声明了ThreadLocalMap这个类，然后在使用ThreadLocal对象set值的时候将当前线程（Thread实例）进行map初始化，并将Threadlocal对应的值塞进map中，下次get的时候，也是使用这个ThreadLcoal的对象（key）去从当前线程的map中获取值（value）就可以了



## 2 ThreadLocalMap的深究-解决冲突

从源码上看，ThreadLocalMap虽然叫做Map，但和我们常规理解的Map不太一样，因为这个类并没有实现Map这个接口，只是定义在ThreadLocal中的一个静态内部类。只是因为在存储的时候也是以key-value的形式作为方法的入参暴露出去，所以称为map。

```
static class ThreadLocalMap {...}
ThreadLocalMap的创建，在使用ThreadLocal对象set值的时候，会创建ThreadLocalMap的对象，可以看到，入参就是KV，key是ThreadLocal对象，value是一个Entry对象，存储kv（HashMap是使用Node作为KV对象存储）。Entry的key是ThreadLocal对象，vaule是set进去的具体值。
 void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

继续看看在创建ThreadLocalMap实例的时候做了什么？其实ThreadLocalMap存储是一个Entry类型的数组，key提供了hashcode用来计算存储的数组地址（散列法解决冲突）
创建Entry数组（初始容量16）
然后获取到key（ThreadLocal对象）的hashcode(是一个自增的原子int型)

```
private final int threadLocalHashCode = nextHashCode();

private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
private static AtomicInteger nextHashCode =    new AtomicInteger();
```

使用**【hashcode 模(%) 数组长度】**的方式得到要将key存储到数组的哪一位。
设置数组的扩容阈值，用以后续扩容

```
ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```

创建ThreadLcoalMap对象只有在当前线程第一次插入kv的时候发生，如果是第二次插入kv，则会进行第三步

这个set的过程其实就是根据ThreadLocal的hashcode来计算存储在Entry数组的位置
利用ThreadLocal的【hashcode 模(%) 数组长度】的方式获取存储在数组的位置
如果当前位置已存在值，则向右移一位，如果也存在值，则继续右移，直到有空位置出现为止
将当前的value存储上面两部得到的索引位置（上面这两步就是**散列法**的实现）
校验是否扩容，如果当前数组的中存储的值得数量大于阈值（数组长度的2/3），则扩容一倍，并将原来的数组的值重新hash至新数组中（这个过程其实就是HashMap的扩容过程）

```
private void set(ThreadLocal key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```



### 散列

![小傅哥 & threadLocal 数据结构](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801223039.png)

如上图是 `ThreadLocal` 存放数据的底层数据结构，包括知识点如下；

1. 它是一个数组结构。
2. `Entry`，这里没用再打开，其实它是一个弱引用实现，`static class Entry extends WeakReference<ThreadLocal<?>>`。这说明只要没用强引用存在，发生GC时就会被垃圾回收。
3. 数据元素采用哈希散列方式进行存储，不过这里的散列使用的是 `斐波那契（Fibonacci）散列法`，后面会具体分析。
4. 另外由于这里不同于HashMap的数据结构，发生哈希碰撞不会存成链表或红黑树，而是使用拉链法进行存储。也就是同一个下标位置发生冲突时，则`+1向后寻址`，直到找到空位置或垃圾回收位置进行存储。





## 3 ThreadLocalMap和HashMap的比较

上述的整个过程其实和HashMap的实现方式很相像，相同点：
两个map都是最终用数组作为存储结构，使用key做索引，value是真正存储在数组索引上的值。

**不同点：解决key冲突的方式**
map解决冲突的方式不一样，HashMap采用链表法，ThreadLocalMap采用散列法（又称开放地址法）
思考：为什么不采用HashMap作为ThreadLocal的存储结构？
个人理解：

引入链表，徒增了数据结构的复杂度，并且链表的读取效率较低
更加灵活。包括方法的定义和数组的管理，更加适合当前场景
不需要HashMap的额外的很多方法和变量，需要一个更加纯粹和干净map，来存储自己需要的值，减少内存的损耗。

## 4 ThreadLocal的生命周期

ThreadLocal的生命周期和当前Thread的生命周期强绑定

### 正常情况

正常情况下（当然会有非正常情况），在线程退出的时候会将threadLocals这个变量置为null，等待JVM去自动回收。
注意：Thread这个方法只是用以系统能够显示的调用退出线程，线程在结束的时候是不会调用这个方法，启动的线程是非守护线程，会在线程结束的时候由jvm自动进行空间的释放和回收。

```
private void exit() {
        if (group != null) {
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
    }
```

### 非正常情况

由于现在多线程一般都是由线程池管理，而线程池的线程一般都是复用的，这样会导致线程一直存活，而如果使用ThreadLocal大量存储变量，会使得空间开始膨胀

### 启发

需要自己来管理ThreadLocal的生命周期，在ThreadLocal使用结束以后及时调用remove（）方法进行清理。



# 源码解读

## 1 初始化

```
new ThreadLocal<>()
```

初始化的过程也很简单，可以按照自己需要的泛型进行设置。但在 `ThreadLocal` 的源码中有一点非常重要，就是获取 `threadLocal` 的哈希值的获取，`threadLocalHashCode`。

```
private final int threadLocalHashCode = nextHashCode();

/**
 * Returns the next hash code.
 */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

如源码中，只要实例化一个 `ThreadLocal` ，就会获取一个相应的哈希值，则例我们做一个例子。

```
@Test
public void test_threadLocalHashCode() throws Exception {
    for (int i = 0; i < 5; i++) {
        ThreadLocal<Object> objectThreadLocal = new ThreadLocal<>();
        Field threadLocalHashCode = objectThreadLocal.getClass().getDeclaredField("threadLocalHashCode");
        threadLocalHashCode.setAccessible(true);
        System.out.println("objectThreadLocal：" + threadLocalHashCode.get(objectThreadLocal));
    }
}
```

因为 `threadLocalHashCode` ，是一个私有属性，所以我们实例化后通过上面的方式进行获取哈希值。

```
objectThreadLocal：-1401181199
objectThreadLocal：239350328
objectThreadLocal：1879881855
objectThreadLocal：-774553914
objectThreadLocal：865977613

Process finished with exit code 0
```

这个值的获取，也就是计算 `ThreadLocalMap`，存储数据时，`ThreadLocal` 的数组下标。只要是这同一个对象，在`set`、`get`时，就可以设置和获取对应的值。

## 2 设置元素

### 2.1 流程图解

```
new ThreadLocal<>().set("小傅哥");
```

设置元素的方法，也就这么一句代码。但设置元素的流程却涉及的比较多，在详细分析代码前，我们先来看一张设置元素的流程图，从图中先了解不同情况的流程之后再对比着学习源码。流程图如下；

![小傅哥 & 设置元素流程图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801223319.png)

乍一看可能感觉有点晕，我们从左往右看，分别有如下知识点；

1. 中间是 `ThreadLocal` 的数组结构，之后在设置元素时分为四种不同的情况，另外元素的插入是通过斐波那契散列计算下标值，进行存放的。
2. 情况1，待插入的下标，是空位置直接插入。
3. 情况2，待插入的下标，不为空，key 相同，直接更新
4. 情况3，待插入的下标，不为空，key 不相同，拉链法寻址
5. 情况4，不为空，key 不相同，碰到过期key。其实情况4，遇到的是弱引用发生GC时，产生的情况。碰到这种情况，`ThreadLocal` 会进行探测清理过期key，这部分清理内容后续讲解。

### 2.2 源码分析

```
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        if (k == key) {
            e.value = value;
            return;
        }
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

在有了上面的图解流程，再看代码部分就比较容易理解了，与之对应的内容包括，如下；

1. `key.threadLocalHashCode & (len-1);`，斐波那契散列，计算数组下标。
2. `Entry`，是一个弱引用对象的实现类，`static class Entry extends WeakReference<ThreadLocal<?>>`，所以在没有外部强引用下，会发生GC，删除key。
3. for循环判断元素是否存在，当前下标不存在元素时，直接设置元素 `tab[i] = new Entry(key, value);`。
4. 如果元素存在，则会判断是否key值相等 `if (k == key)`，相等则更新值。
5. 如果不相等，就到了我们的 `replaceStaleEntry`，也就是上图说到的探测式清理过期元素。

**综上**，就是元素存放的全部过程，整体结构的设计方式非常赞👍，极大的利用了散列效果，也把弱引用使用的非常6！

## 3 扩容机制

### 3.1 扩容条件

```
只要使用到数组结构，就一定会有扩容
if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
```

在我们阅读设置元素时，有以上这么一块代码，判断是否扩容。

- 首先，进行`启发式清理*cleanSomeSlots*`，把过期元素清理掉，看空间是否
- 之后，判断`sz >= threshold`，其中 `threshold = len * 2 / 3`，也就是说数组中天填充的元素，大于 `len * 2 / 3`，就需要扩容了。
- 最后，就是我们要分析的重点，`rehash();`，扩容重新计算元素位置。

### 3.2 源码分析

**探测式清理和校验**

```
private void rehash() {
    expungeStaleEntries();
    
    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}

private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

- 这部分是主要是探测式清理过期元素，以及判断清理后是否满足扩容条件，size >= threshold * 3/4
- 满足后执行扩容操作，其实扩容完的核心操作就是重新计算哈希值，把元素填充到新的数组中。

**rehash() 扩容**

```
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

**以上**，代码就是扩容的整体操作，具体包括如下步骤；

1. 首先把数组长度扩容到原来的2倍，`oldLen * 2`，实例化新数组。
2. 遍历for，所有的旧数组中的元素，重新放到新数组中。
3. 在放置数组的过程中，如果发生哈希碰撞，则链式法顺延。
4. 同时这还有检测key值的操作 `if (k == null)`，方便GC。

## 4 获取元素

### 4.1 流程图解

```
new ThreadLocal<>().get();
```

同样获取元素也就这么一句代码，如果没有分析源码之前，你能考虑到它在不同的数据结构下，获取元素时候都做了什么操作吗。我们先来看下图，分为如下种情况；

![小傅哥 & 获取元素图解](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801223456.png)

按照不同的数据元素存储情况，基本包括如下情况；

1. 直接定位到，没有哈希冲突，直接返回元素即可。
2. 没有直接定位到了，key不同，需要拉链式寻找。
3. 没有直接定位到了，key不同，拉链式寻找，遇到GC清理元素，需要探测式清理，再寻找元素。

### 4.2 源码分析

```
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

**好了**，这部分就是获取元素的源码部分，和我们图中列举的情况是一致的。`expungeStaleEntry`，是发现有 `key == null` 时，进行清理过期元素，并把后续位置的元素，前移。

## 5 元素清理

### 5.1 探测式清理[expungeStaleEntry]

探测式清理，是以当前遇到的 GC 元素开始，向后不断的清理。直到遇到 null 为止，才停止 rehash 计算`Rehash until we encounter null`。

**expungeStaleEntry**

```
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

**以上**，探测式清理在获取元素中使用到； `new ThreadLocal<>().get() -> map.getEntry(this) -> getEntryAfterMiss(key, i, e) -> expungeStaleEntry(i)`

### 5.2 启发式清理[cleanSomeSlots]

```
Heuristically scan some cells looking for stale entries.
This is invoked when either a new element is added, or
another stale one has been expunged. It performs a
logarithmic number of scans, as a balance between no
scanning (fast but retains garbage) and a number of scans
proportional to number of elements, that would find all
garbage but would cause some insertions to take O(n) time.
```

**启发式清理**，有这么一段注释，大概意思是；试探的扫描一些单元格，寻找过期元素，也就是被垃圾回收的元素。*当添加新元素或删除另一个过时元素时，将调用此函数。它执行对数扫描次数，作为不扫描（快速但保留垃圾）和与元素数量成比例的扫描次数之间的平衡，这将找到所有垃圾，但会导致一些插入花费O（n）时间。*

```
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

while 循环中不断的右移进行寻找需要被清理的过期元素，最终都会使用 `expungeStaleEntry` 进行处理，这里还包括元素的移位。



# 应用场景

在介绍具体的使用场景之前，我们先来抽象一下：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801223705.png)

这个图表示：多个线程的生命周期不同，当一个线程在其生命周期内的某个时候，调用ThreadLocal.set()方法，其实就在该线程内部启用了一个局部变量，而后这个局部变量可以在该线程生命周期的任何时候被获取，直到调用ThreadLocal.remove()方法或者线程消亡。

线程通过**ThreadLocal**提供的接口来操作自己内部的映射表，或者可以在语意上这么理解：线程把**ThreadLocal**当做自己的局部变量，不过对这个变量的赋值操作是set()，读取操作是get()，清空操作是remove()



## 1 Android Looper

Android中有一个很常见的操作：使用Handler将消息抛送到线程的消息队列。控制消息队列的类是**Looper**，每个拥有消息队列的线程，都会有一个独立的**Looper**类，用于处理本线程的消息。 一种实现方式是：在线程类中，声明一个**Looper**类型的局部变量，当线程运行起来时，创建**Looper**对象，并开始进行无限循环，代码示意如下：

```
public class LooperThread extends Thread {
    private Looper mLooper;

    @Override
    public void run() {
        // 创建Looper对象(实际上，Looper类的构造器是私有的)
        mLooper = new Looper();
        // 开始无限循环处理消息
        mLooper.loop();
    }

    public Looper getLooper() {
        return mLooper;
    }
}
```

**注意到**，这种实现方式需要增加一个方法：**getLooper()**，因为其他线程可能需要获取**LooperThread**的消息队列。 然而，Android并不是采用的上述实现方式，而是利用**ThreadLocal**来保存**Looper**对象，当一个线程想要拥有消息队列时，调用**Looper.prepare()**方法便可完成消息队列的初始化，然后调用**Looper.loop()**便会开始无限循环，不断从消息队列上取出消息进行处理。先来看**Looper**的代码实现片段：

```
public final class Looper {
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    // 私有构造器，意味着外部不能调用
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        // 通过ThreadLocal保存新建的Looper对象
        sThreadLocal.set(new Looper(quitAllowed));
    }

    public static Looper myLooper() {
        // 返回实际线程的Looper对象
        return sThreadLocal.get();
    }
}
```

**Looper**中定义了一个静态变量*`sThreadLocal`*，构造器都是私有的(private)，即外部无法调用，然后提供了一个**prepare()**方法，当该方法被调用时，便往*`sThreadLocal`*中设置一个**Looper**对象。

上文剖析过**ThreadLocal**的实现，可以知道：哪个线程调用了**prepare()**方法，**Looper**对象就添加到了那个具体线程的**ThreadLocalMap**映射表中，表中每一项的*Key*是*`sLocalThread`*，*Value*是**Looper**对象，这样一来，就等价于线程拥有了**Looper**这个局部变量。如何获取线程中的**Looper**对象呢？在线程中直接调用**ThreadLocal.get()**方法就可以了，所以**Looper**类封装了一个静态方法**myLooper()**，做的就是获取当前线程**Looper**对象的买卖。

Android中，真正带消息队列的线程实现是**HandlerThread**，与上文中模拟的**LooperThread**的实现方式如出一辙，不过是利用了**ThreadLocal**这个编程工具：

```
public class HandlerThread extends Thread {
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();  // 初始化消息队列，即将Looper对象添加到实际线程的ThreadLocalMap中
        synchronized (this) {
            mLooper = Looper.myLooper(); // 获取实际线程的Looper对象
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();  // 开始无限循环处理消息
        mTid = -1;
    }

    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
}
```

当线程运行起来时，往**ThreadLocal**中添加了一个**Looper**对象，然后开始无限循环处理消息。往**ThreadLocal**中添加对象的行为，就意味着这个对象是属于每个线程的局部变量。

当有多个HandlerThread同时运行时，它们的关系如下图所示：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801223758.png)

每一个HandlerThread线程内部都有**Key-Value Pairs**，*Value*是不同的Looper对象，而*Key*是指向同一个静态ThreadLocal对象的弱引用。

## 2 Android SQLiteDatabase

Android中进行数据库的事务操作时，通常都会在某个工作线程中调用SQLiteDatabase.beginTransaction()方法，然后开始具体的数据库操作。有些时候，并发操作数据库的线程会存在多个，要操作数据库，是要发起连接的，Android封装了一个类**SQLiteSession**，专门来管理数据库连接，每个线程都需要**SQLiteSession**对象，那线程怎样才能获取到一个独立的**SQLiteSession**对象呢？这种场景下，便有了ThreadLocal的用武之地了。

```
public final class SQLiteDatabase extends SQLiteClosable {
    // 定义ThreadLocal，存储的对象类型是SQLiteSession
    private final ThreadLocal<SQLiteSession> mThreadSession = new ThreadLocal<SQLiteSession>() {
        @Override
        protected SQLiteSession initialValue() {
            return createSession();
        }
    };

    SQLiteSession getThreadSession() {
        // 通过ThreadLocal获取SQLiteSession对象
        return mThreadSession.get(); // initialValue() throws if database closed
    }

    private void beginTransaction(SQLiteTransactionListener transactionListener,
            boolean exclusive) {
        acquireReference();
        try {
            // 获取SQLiteSession对象后，开始数据库的事务操作
            getThreadSession().beginTransaction(
                    exclusive ? SQLiteSession.TRANSACTION_MODE_EXCLUSIVE :
                            SQLiteSession.TRANSACTION_MODE_IMMEDIATE,
                    transactionListener,
                    getThreadDefaultConnectionFlags(false /*readOnly*/), null);
        } finally {
            releaseReference();
        }
    }

}
```

SQLiteDatabase中定义了**ThreadLocal**，所存储对象的类型是SQLiteSession。每当在线程中调用**SQLiteDatabase.beginTransaction()**方法时，表示要开始数据库的事务操作了，这时候会先从**ThreadLocal**中取出属于当前线程的SQLiteSession对象。

在多进程多线程访问数据库的情况下，它们的关系图如下所示：

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801223737.png)







## 每个线程维护了一个“序列号”

```java
public class SerialNum {
    // The next serial number to be assigned
    private static int nextSerialNum = 0;

    private static ThreadLocal serialNum = new ThreadLocal() {
        protected synchronized Object initialValue() {
            return new Integer(nextSerialNum++);
        }
    };

    public static int get() {
        return ((Integer) (serialNum.get())).intValue();
    }
}
    
```



## Session的管理

经典的另外一个例子：

```java
private static final ThreadLocal threadSession = new ThreadLocal();  
  
public static Session getSession() throws InfrastructureException {  
    Session s = (Session) threadSession.get();  
    try {  
        if (s == null) {  
            s = getSessionFactory().openSession();  
            threadSession.set(s);  
        }  
    } catch (HibernateException ex) {  
        throw new InfrastructureException(ex);  
    }  
    return s;  
}      
```

## 在线程内部创建ThreadLocal

还有一种用法是在线程类内部创建ThreadLocal，基本步骤如下：

- 在多线程的类(如ThreadDemo类)中，创建一个ThreadLocal对象threadXxx，用来保存线程间需要隔离处理的对象xxx。
- 在ThreadDemo类中，创建一个获取要隔离访问的数据的方法getXxx()，在方法中判断，若ThreadLocal对象为null时候，应该new()一个隔离访问类型的对象，并强制转换为要应用的类型。
- 在ThreadDemo类的run()方法中，通过调用getXxx()方法获取要操作的数据，这样可以保证每个线程对应一个数据对象，在任何时刻都操作的是这个对象。

```java
public class ThreadLocalTest implements Runnable{
    
    ThreadLocal<Student> StudentThreadLocal = new ThreadLocal<Student>();

    @Override
    public void run() {
        String currentThreadName = Thread.currentThread().getName();
        System.out.println(currentThreadName + " is running...");
        Random random = new Random();
        int age = random.nextInt(100);
        System.out.println(currentThreadName + " is set age: "  + age);
        Student Student = getStudentt(); //通过这个方法，为每个线程都独立的new一个Studentt对象，每个线程的的Studentt对象都可以设置不同的值
        Student.setAge(age);
        System.out.println(currentThreadName + " is first get age: " + Student.getAge());
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println( currentThreadName + " is second get age: " + Student.getAge());
        
    }
    
    private Student getStudentt() {
        Student Student = StudentThreadLocal.get();
        if (null == Student) {
            Student = new Student();
            StudentThreadLocal.set(Student);
        }
        return Student;
    }

    public static void main(String[] args) {
        ThreadLocalTest t = new ThreadLocalTest();
        Thread t1 = new Thread(t,"Thread A");
        Thread t2 = new Thread(t,"Thread B");
        t1.start();
        t2.start();
    }
    
}

class Student{
    int age;
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
    
}    
```

## java 开发手册中推荐的 ThreadLocal

看看阿里巴巴 java 开发手册中推荐的 ThreadLocal 的用法:

```java
import java.text.DateFormat;
import java.text.SimpleDateFormat;
 
public class DateUtils {
    public static final ThreadLocal<DateFormat> threadLocal = new ThreadLocal<DateFormat>(){
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };
}
    
```



然后我们再要用到 DateFormat 对象的地方，这样调用：

```java
DateUtils.df.get().format(new Date());    
```



### SimpleDateFormat

```
private SimpleDateFormat f = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

public void seckillSku(){
    String dateStr = f.format(new Date());
    // 业务流程
}
```

你写过这样的代码吗？如果还在这么写，那就已经犯了一个线程安全的错误。`SimpleDateFormat`，并不是一个线程安全的类。

**线程不安全验证**

```
private static SimpleDateFormat f = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

public static void main(String[] args) {
    while (true) {
        new Thread(() -> {
            String dateStr = f.format(new Date());
            try {
                Date parseDate = f.parse(dateStr);
                String dateStrCheck = f.format(parseDate);
                boolean equals = dateStr.equals(dateStrCheck);
                if (!equals) {
                    System.out.println(equals + " " + dateStr + " " + dateStrCheck);
                } else {
                    System.out.println(equals);
                }
            } catch (ParseException e) {
                System.out.println(e.getMessage());
            }
        }).start();
    }
}
```

这是一个多线程下 `SimpleDateFormat` 的验证代码。当 `equals 为false` 时，证明线程不安全。运行结果如下；

```
true
true
false 2020-09-23 11:40:42 2230-09-23 11:40:42
true
true
false 2020-09-23 11:40:42 2020-09-23 11:40:00
false 2020-09-23 11:40:42 2020-09-23 11:40:00
false 2020-09-23 11:40:00 2020-09-23 11:40:42
true
false 2020-09-23 11:40:42 2020-08-31 11:40:42
true
```



## 链路追踪









# 面试题

## 什么是ThreadLocal? 用来解决什么问题的?

## 说说你对ThreadLocal的理解

## ThreadLocal是如何实现线程隔离的?

## 为什么ThreadLocal会造成内存泄露? 如何解决

## 还有哪些使用ThreadLocal的应用场景?





# 参考

https://mp.weixin.qq.com/s/WcEbLtegeFOplIhjgn6QdA

https://bugstack.cn/interview/2020/09/23/%E9%9D%A2%E7%BB%8F%E6%89%8B%E5%86%8C-%E7%AC%AC12%E7%AF%87-%E9%9D%A2%E8%AF%95%E5%AE%98-ThreadLocal-%E4%BD%A0%E8%A6%81%E8%BF%99%E4%B9%88%E9%97%AE-%E6%88%91%E5%B0%B1%E6%8C%82%E4%BA%86.html

https://duanqz.github.io/2018-03-15-Java-ThreadLocal#11-%E5%88%9D%E5%A7%8B%E5%BD%A2%E6%80%81