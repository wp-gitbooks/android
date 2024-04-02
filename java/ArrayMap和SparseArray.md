# 概述

ArrayMap是Android专门针对**内存优化**而设计的，用于取代Java API中的HashMap数据结构。

为了更进一步优化key是int类型的Map，Android再次提供效率更高的数据结构**SparseArray**，可避免自动装箱过程。

对于key为其他类型则可使用**ArrayMap**。HashMap的查找和插入时间复杂度为O(1)的代价是牺牲大量的内存来实现的，而SparseArray和ArrayMap性能略逊于HashMap，但更节省内存。


>SparseArray有两个优点：
>1.避免了**自动装箱**（auto-boxing），
>2.数据结构不会依赖于外部对象映射。我们知道HashMap 采用一种所谓的“Hash 算法”来决定每个元素的存储位置，存放的都是数组元素的引用，通过每个对象的hash值来映射对象。而SparseArray则是用数组数据结构来保存映射，然后通过**折半查找来找到对象**。但其实一般来说，SparseArray执行效率比HashMap要慢一点，因为查找需要折半查找，而添加删除则需要在数组中执行，而HashMap都是通过外部映射。但相对来说影响不大，最主要是SparseArray不需要开辟内存空间来额外存储外部映射，从而节省内存。


# 为何HashMap占用内存较大？

为何`SparseArray`会比`HashMap`更节省内存，这要从它们各自的结构说起。HashMap底层数据结构是一个 **数组+链表** 的组合（关于数组和链表的概念，这里就不多阐述了），它采用一种所谓的“Hash 算法”来决定每个元素的存储位置。当程序执行 `map.put(key,Obect)` 方法 时，系统将调用key对象的 `hashCode()` 方法得到其 hashCode 值（每个Java对象都有 hashCode() 方法，都可通过该方法获得它的 hashCode 值）。得到这个对象的 hashCode 值之后，系统会根据该 hashCode 值再hash一遍来决定该元素在数组中的存储位置。

但是这就存在一个问题，如果两个key算出来的hash值刚好相等，也就是存放的数组位置一样时，就产生了Hash冲突（因为原本数组的那个位置已经有一个元素存放着，而一个位置只能存放一组数据），那HashMap是怎么解决这种冲突的呢？
 HashMap采用**链表法**来解决Hash冲突，也就是说，如果发生这种情况，HashMap会在数组中冲突的那个位置，将后加入的元素指向原来占有数组位置的那个元素，从而追加形成一个链表：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210804161556)


 HashMap中初始的存储大小就是一个容量为16的数组，所以当我们创建出一个HashMap对象时，即使里面没有不论什么元素。也要分别一块内存空间给它，并且，我们再不断的向HashMap里put数据时，当达到一定的容量限制时，HashMap的空间将会扩大为原来的2倍，所以HashMap是比较占内存的。


# 为何 SparseArray 更为优化？

先了解一个基本概念——**什么是自动装箱？**
自动装箱就是指自动将基本数据类型转换为包装器类型，比如下面这句代码：

```undefined
Integer i = 99;
```

99是基本数据类型，将它直接赋值给Integer类型对象i时，就会自动将我们的基本类型int包装成Integer。装箱操作会创建对象，频繁的装箱操作会消耗许多内存，影响性能。

而SparseArray又称为稀疏数组，与HashMap不同，其内部是直接通过维护两个数组来实现存储：

```java
public class SparseArray<E> implements Cloneable {

    private int[] mKeys;
    private Object[] mValues;
    ...
}
```

可以看到，一组存储键，一组存储值，key数组的类型是int型，也就是说，**SparseArray只支持key为int类型的数据存储**，关键就在这里，由于它是直接维护了一个int数组，那么key就避免了自动装箱的过程，举个例子，比如我们用HashMap存储下面这组数据：

```dart
HashMap<Integer, String> hashMap = new HashMap<>();
hashMap.put(1, "test");
hashMap.put(2, "test");
hashMap.put(3, "test");
```

每次put进去的时候，由于传进去的是1，2，3，都是int基本类型，HashMap会自动帮我们包装成Integer类型的对象（也就是刚说的自动装箱），那么就肯定会消耗更多内存。但如果是SparseArray来存储的话，就直接将key存储在key数组了，省去了装箱这个过程，从而节省了内存开销。

另一方面，对SparseArray增删查改操作时，其内部会不断检查回收无用空间，从而压缩占用的内存大小，我们看下它的`put`方法：

```java
public void put(int key, E value) {
        //先调用二分法查询该key在数组中的位置
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            //大于0说明已经存在数组中，可以直接赋值
            mValues[i] = value;
        } else {
            //小于0说明这是一个新的键值对，且它应该插在数组中的第i个位置
            i = ~i;
            //根据DELETED来查询当前位置的值是否已经被删除
            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }
            //如果当前容量已满
            if (mGarbage && mSize >= mKeys.length) {
                //回收无效空间
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }

            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
}
```

可以看到，`SparseArray`会先调用二分法去查询key应该存放在数组中的位置，所以`SparseArray`的key数组一定是有序排列的，然后会用一个`DELETED`来作为当前位置的元素是否已经被删除，`DELETED`会在调用`remove`移除元素的时候赋给对应位置Value，如下：


```java
public void remove(int key) {
    delete(key);
}

public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}
```

`SparseArray`通过这个来作为它压缩空间的一个标志（即该位置可不可以被回收），这样子也进一步节省了空间。

从刚才可以看出，无论是SparseArray的`put`还是`delete`（其实其他操作比如`get`也都是通过二分法寻找下标），都是通过二分法去查询这个key应该被存放的位置。而HashMap在插入的时候，不需要去遍历整个集合，而是直接通过hash计算出位置插入。所以在插入效率上，SparseArray会比HashMap稍慢一些，但在数据量不大的情况下，两者的差别不大。


# 结语

`SparseArray`与`HashMap`相比，**最大的优势在于内存方面**，无论数据量级大小如何，`SparseArray`所占用的内存都会比HashMap小，在Android中内存是极为重要的，所以在需要保存<Integer,Object>键值对的场景中，推荐使用`SparseArray`替换`HashMap`。换句话说，`SparseArray`是Android中为<Integer,Object>这样的HashMap专门写的类，它避开了**自动装箱并且压缩稀疏数组**，目的就是为了节省内存。
 另外，Android还提供了其他几种类似的集合类：`SparseIntArray`、`SparseBooleanArray`、`SparseLongArray`，可以支持存储<Integer,Integer>、<Integer,Boolean>、<Integer,Long>的数据类型，也就是同时让Value也避开了装箱过程，进一步优化。



# 参考

https://www.bilibili.com/video/BV1ff4y1a7PM?p=7


http://gityuan.com/2019/01/13/arraymap/

https://zhuanlan.zhihu.com/p/152230430

https://juejin.cn/post/6844903920486072328#heading-0

https://jiezhi.github.io/2017/03/30/android-arraymap-vs-sparsearray/

https://juejin.cn/post/6844903442901630983#heading-15

