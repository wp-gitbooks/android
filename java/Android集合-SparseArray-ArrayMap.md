# 一.前言

SparseArray 和 ArrayMap 是 Android 系统 api 中用于存储键值对数据的集合,相比于 java 集合的 HashMap ,SparseArray 和 ArrayMap 在某些场景下能够以时间换空间策略,带来内存上效率的提升,因此更适合移动设备。

# 二.SparseArray

## 1简介

```text
/**
 * SparseArrays map integers to Objects.  Unlike a normal array of Objects,
 * there can be gaps in the indices.  It is intended to be more memory efficient
 * than using a HashMap to map Integers to Objects, both because it avoids
 * auto-boxing keys and its data structure doesn't rely on an extra entry object
 * for each mapping.
 *
 * <p>Note that this container keeps its mappings in an array data structure,
 * using a binary search to find keys.  The implementation is not intended to be appropriate for
 * data structures
 * that may contain large numbers of items.  It is generally slower than a traditional
 * HashMap, since lookups require a binary search and adds and removes require inserting
 * and deleting entries in the array.  For containers holding up to hundreds of items,
 * the performance difference is not significant, less than 50%.</p>
 *
 * <p>To help with performance, the container includes an optimization when removing
 * keys: instead of compacting its array immediately, it leaves the removed entry marked
 * as deleted.  The entry can then be re-used for the same key, or compacted later in
 * a single garbage collection step of all removed entries.  This garbage collection will
 * need to be performed at any time the array needs to be grown or the the map size or
 * entry values are retrieved.</p>
 *
 * <p>It is possible to iterate over the items in this container using
 * {@link #keyAt(int)} and {@link #valueAt(int)}. Iterating over the keys using
 * <code>keyAt(int)</code> with ascending values of the index will return the
 * keys in ascending order, or the values corresponding to the keys in ascending
 * order in the case of <code>valueAt(int)</code>.</p>
 */
  public class SparseArray<E> implements Cloneable {
      ...
  }
```

在源码中对 SparseArray 的介绍就如上面的注释,从中就可以知道以下几点：

### (1).

SparseArray **存储 int -> Objects 映射关系**（不是 Integer ）,类似的类还有 SparseBooleanArray ，SparseIntArray ， SparseLongArray ，LongSparseArray。对应的映射关系如下：

类 | 映射关系 ---|--- SparseArray | int -> Objects SparseBooleanArray | int -> boolean SparseIntArray | int -> int SparseLongArray | int -> long LongSparseArray | long -> Objects

### (2).

不像其他的数组结构下标是连续的，它能够允许某些下标不存在，因此称为 **SparseArray (稀疏数组)**,因为避免了自动装拆箱,且不用创建其他的实体（HashMap 需要创建 Entry），因此在内存上有更高的效率。

### (3).

与 HashMap 中使用 hash 值定位下标的方式不同，SparseArray 使用的是**二分查找的方式**,因此当有大量数据的时候，SparseArray 的速度将明显慢于 HashMap

### (4).

在删除的元素的时候，SparseArray 不是立即对数组进行压缩（去除数组中为 空的元素），而是使用一个标志，对**删除的位置进行标志，只有在数组扩容或者 键值对数量和某个元素需要恢复的时候，才会统一进行压缩**。

## 2.源码

```text
public class SparseArray<E> implements Cloneable {
    private static final Object DELETED = new Object(); //删除的标志
    private boolean mGarbage = false; //压缩标志

    private int[] mKeys; //int 类型的数组，存储 key
    private Object[] mValues; //Object 类型的数组，存储 value
    private int mSize; // 键值对的数量

     //默认构造器，初始化 数量为 10 
    public SparseArray() {
        this(10);
    }

   //指定数量的构造器
    public SparseArray(int initialCapacity) {
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0;
    }
```

### 1.put 方法

```text
public void put(int key, E value) {
        //先二分查找对应的下标
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        ...
    }
```

put 方法首先要就要查找对应的需要插入的位置，因为 SparseArray 是一个稀疏数组即可能有这样的情况 （0 ，1 ，3 ，5，6） 因此首先就要找到对应的插入位置。二分查找对应的算法如下

```text
static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }
```

当查找成功就返回对应的下标，但是查找失败的时候注意最后返回的是 ~lo,而不是简单的 -1, 其实 ~lo 对应的就是需要插入的位置，比如 （0 ，1 ，3 ，5，6）查找 2，返回的就是 -2 ，取反可以将正数转为负数，如果是负数就说明查找失败，比如返回 -2 就说明查找失败，插入的位置为 2 .

接着回到 put 方法。

```text
public void put(int key, E value) {
        //使用二分查找
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        //i>0
        //说明查找成功，直接更新
        if (i >= 0) {
            mValues[i] = value;
        //查找失败的话
        } else {
        //取反获取对应的插入的位置
            i = ~i;

            //如果下标 小于 元素数量并且之前被标志位删除的话
            //重新赋值即可
            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }

           //否则判断是否需要进行数组压缩
           //需要的话就调用 gc 方法进行压缩
           //并重新二分查找获取下标
            if (mGarbage && mSize >= mKeys.length) {
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }

            //对数组进行扩容 并复制
            //GrowingArrayUtils.insert 使用的是  System.arraycopy
            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
    }
```

gc 压缩数组的源码如下

```text
private void gc() {
        // Log.e("SparseArray", "gc start with " + mSize);

        int n = mSize;
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;

        for (int i = 0; i < n; i++) {
            Object val = values[i];

            if (val != DELETED) {
                if (i != o) {
                    keys[o] = keys[i];
                    values[o] = val;
                    values[i] = null;
                }

                o++;
            }
        }

        mGarbage = false;
        mSize = o;

        // Log.e("SparseArray", "gc end with " + mSize);
    }
```



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210822145239.jpg)



下一次进行 arrayCopy 的时候多余的 null 就不会进行复制，延迟删除机并一次性压缩就提高了效率。

### 2.append 方法

put 方法是在数组中插入元素，那么 append 对应的就是在数组末尾追加元素

```text
public void append(int key, E value) {
        //如果 key 小于 存在的key
        //就使用 put 进行插入
        if (mSize != 0 && key <= mKeys[mSize - 1]) {
            put(key, value);
            return;
        }

        //判断是否需要 gc 
        if (mGarbage && mSize >= mKeys.length) {
            gc();
        }

        //调用 GrowingArrayUtils.append 在数组末尾追加元素
        //内部也是使用 System.arrayCopy 
        mKeys = GrowingArrayUtils.append(mKeys, mSize, key);
        mValues = GrowingArrayUtils.append(mValues, mSize, value);
        mSize++;
    }
```

### 3.get 方法

get 方法 方法就相对简单了

```text
public E get(int key) {
        return get(key, null);
    }

    /**
     * Gets the Object mapped from the specified key, or the specified Object
     * if no such mapping has been made.
     */
    @SuppressWarnings("unchecked")
    public E get(int key, E valueIfKeyNotFound) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        //如果查找失败或者元素已经被删除
        //就返回 null 或者默认的值
        if (i < 0 || mValues[i] == DELETED) {
            return valueIfKeyNotFound;
        } else {
            return (E) mValues[i];
        }
    }
```

## 与 HashMap 的对比

优点 - 频繁的插入删除操作效率高,因为**延迟删除标志 DELETE** - 通过 gc 函数来清除，内存利用率高 - **不用自动拆箱**

缺点 - 存取因二分查找的**时间复杂度 O(log n)**，数据较多的情况下，效率没有 HashMap 高 - key 只能是 int 或者 long

因此 SparseArray 应用场景为 **数据较少**，并且 存取的 value 为指定类型的，比如 boolean、int、long，可以避免自动装箱和拆箱问题。

# 三.ArrayMap

```text
/**
 * ArrayMap is a generic key->value mapping data structure that is
 * designed to be more memory efficient than a traditional {@link java.util.HashMap}.
 * It keeps its mappings in an array data structure -- an integer array of hash
 * codes for each item, and an Object array of the key/value pairs.  This allows it to
 * avoid having to create an extra object for every entry put in to the map, and it
 * also tries to control the growth of the size of these arrays more aggressively
 * (since growing them only requires copying the entries in the array, not rebuilding
 * a hash map).
 *
 * <p>Note that this implementation is not intended to be appropriate for data structures
 * that may contain large numbers of items.  It is generally slower than a traditional
 * HashMap, since lookups require a binary search and adds and removes require inserting
 * and deleting entries in the array.  For containers holding up to hundreds of items,
 * the performance difference is not significant, less than 50%.</p>
 *
 * <p>Because this container is intended to better balance memory use, unlike most other
 * standard Java containers it will shrink its array as items are removed from it.  Currently
 * you have no control over this shrinking -- if you set a capacity and then remove an
 * item, it may reduce the capacity to better match the current size.  In the future an
 * explicit call to set the capacity should turn off this aggressive shrinking behavior.</p>
 */
public final class ArrayMap<K, V> implements Map<K, V> {
    ...
}
```

## 1.简介

在源码中对 ArrayMap 的介绍主要关注在第一段，后面的前面关于 SparseArray 的介绍基本一样，这里就不再赘述，只关注第一段。

ArrayMap 是一个简单的集合用于存储 键值对 。它比传统的键值对集合 HashMap 要在内存上更有效率，因为它维持的是一个数组的数据结构，**一个整型数组用于保存每一项的 hash 值**,一个 键值数组保存 键值对。因此这就使得 ArrayMap 不用创建额外的实体（HashMap 中需要创建 Entry ），在扩容的时候只需要复制就行了，不用重新 hash .

对应的数据结构如图。
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210822145930.png)



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210822145410.jpg)



## 2.源码

```text
private static final boolean DEBUG = false;
    private static final String TAG = "ArrayMap";

    //
    private static final boolean CONCURRENT_MODIFICATION_EXCEPTIONS = true;

    //缓存长度为 4 时的数组
    private static final int BASE_SIZE = 4;

    //缓存数组数量的最大值，也就是说最多只能缓存10个数组
    private static final int CACHE_SIZE = 10;

    //空数组
    static final int[] EMPTY_IMMUTABLE_INTS = new int[0];

    //空 ArrayMap
    public static final ArrayMap EMPTY = new ArrayMap<>(-1);

   //缓存数组
   // mBaseCache 用于缓存长度为 4 的数组
   //mBaseCacheSize 记录缓存的个数
    static Object[] mBaseCache;
    static int mBaseCacheSize;

    //mTwiceBaseCache 用于缓存长度为 8 的数组
    //mTwiceBaseCacheSize 记录个数
    static Object[] mTwiceBaseCache;
    static int mTwiceBaseCacheSize;

    //是否需要唯一的 hash 值
    //因为 hashCode 不唯一
    final boolean mIdentityHashCode;
    int[] mHashes; //存放每一项的 hash 值
    Object[] mArray; //key/value 数组
    int mSize; //键值对大小
    MapCollections<K, V> mCollections;
```

构造函数

```text
public ArrayMap() {
        this(0, false);
    }

    public ArrayMap(int capacity) {
        this(capacity, false);
    }

    /** {@hide} */
    public ArrayMap(int capacity, boolean identityHashCode) {
        mIdentityHashCode = identityHashCode;
        // If this is immutable, use the sentinal EMPTY_IMMUTABLE_INTS
        // instance instead of the usual EmptyArray.INT. The reference
        // is checked later to see if the array is allowed to grow.
        if (capacity < 0) {
            mHashes = EMPTY_IMMUTABLE_INTS;
            mArray = EmptyArray.OBJECT;
        } else if (capacity == 0) {
            mHashes = EmptyArray.INT;
            mArray = EmptyArray.OBJECT;
        } else {
            allocArrays(capacity);
        }
        mSize = 0;
    }

    public ArrayMap(ArrayMap<K, V> map) {
        this();
        if (map != null) {
            putAll(map);
        }
    }
```

无论是哪个构造函数，最后都会调用 两个参数的 ArrayMap(int capacity, boolean identityHashCode) - identityHashCode,如果 true 则 hash 值是默认的 hashCode System.identityHashCode(key) 不管 key 的hashCode 是否被重写 。否则就是 key.hashCode() 这个 hashCode 就有可能使重写的情况。

注意最后 如果设置的初始大小大于 0 ，则会调用 allocArrays 。allocArrays 这个方法就是从缓存的数组获取数组，避免重复的创建，既然有获取缓存当然有添加缓存，相应的添加缓存的方法就是 freeArrays 。

### 首先看缓存的数组是如何添加的

```text
static Object[] mBaseCache;
    static int mBaseCacheSize;
    static Object[] mTwiceBaseCache;
    static int mTwiceBaseCacheSize;

    //首先添加缓存
    private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
        //当 hash 数组大小为 8 的时候
        if (hashes.length == (BASE_SIZE*2)) {
            synchronized (ArrayMap.class) {
               //如果缓存的数组的个数小于 CACHE_SIZE 即 10 个

                if (mTwiceBaseCacheSize < CACHE_SIZE) {
                   //就将 旧的数组第一个元素设置为 mTwiceBaseCache
                    array[0] = mTwiceBaseCache;

                   /第二个元素设置为 旧的 hash 数组
                    array[1] = hashes;

                    //其他元素设置为 null
                    for (int i=(size<<1)-1; i>=2; i--) {
                        array[i] = null;
                    }

                    //把 mTwiceBaseCache 指向就得数组
                    mTwiceBaseCache = array;
                    mTwiceBaseCacheSize++;
                    if (DEBUG) Log.d(TAG, "Storing 2x cache " + array
                            + " now have " + mTwiceBaseCacheSize + " entries");
                }
            }
        //当 hash 数组的大小为 4 的时候
        //情况和上述的一样
        } else if (hashes.length == BASE_SIZE) {
            synchronized (ArrayMap.class) {
                if (mBaseCacheSize < CACHE_SIZE) {
                    array[0] = mBaseCache;
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        array[i] = null;
                    }
                    mBaseCache = array;
                    mBaseCacheSize++;
                    if (DEBUG) Log.d(TAG, "Storing 1x cache " + array
                            + " now have " + mBaseCacheSize + " entries");
                }
            }
        }
    }
```

注意到能够添加到缓存的条件只有两种情况 hash 数组的长度为 4 或者 8 添加的情况如图：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210822145500.jpg)



可以看出这是一个类似链表的结构，相应的获取缓存就是取出链表的头节点，并设置新的头节点

```text
//获取缓存的方法
private void allocArrays(final int size) {

        if (mHashes == EMPTY_IMMUTABLE_INTS) {
            throw new UnsupportedOperationException("ArrayMap is immutable");
        }
        if (size == (BASE_SIZE*2)) {
            synchronized (ArrayMap.class) {
                if (mTwiceBaseCache != null) {
                   //获取头结点数组
                    final Object[] array = mTwiceBaseCache;
                    mArray = array;
                    //设置新的头节点
                    mTwiceBaseCache = (Object[])array[0];
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;
                    //缓存数减 1 
                    mTwiceBaseCacheSize--;
                    if (DEBUG) Log.d(TAG, "Retrieving 2x cache " + mHashes
                            + " now have " + mTwiceBaseCacheSize + " entries");
                    return;
                }
            }
        } else if (size == BASE_SIZE) {
            synchronized (ArrayMap.class) {
                if (mBaseCache != null) {
                    final Object[] array = mBaseCache;
                    mArray = array;
                    mBaseCache = (Object[])array[0];
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;
                    mBaseCacheSize--;
                    if (DEBUG) Log.d(TAG, "Retrieving 1x cache " + mHashes
                            + " now have " + mBaseCacheSize + " entries");
                    return;
                }
            }
        }

       //如果需要的长度既不是 4 也不是 8 就创建新的数组。

        mHashes = new int[size];
        mArray = new Object[size<<1];
    }
```

### put 方法

```text
public V put(K key, V value) {
        final int osize = mSize;
        final int hash;
        int index;
        // key 允许为 null 
        //对应下标为 0 
        if (key == null) {
            hash = 0;
            index = indexOfNull();
        //根据 hash 值获取下标
        //这个过程 主要由 indexOf 完成
        } else {
            hash = mIdentityHashCode ? System.identityHashCode(key) : key.hashCode();
            index = indexOf(key, hash);
        }
        //如果下标存在就 设置并返回
        if (index >= 0) {
            index = (index<<1) + 1;
            final V old = (V)mArray[index];
            mArray[index] = value;
            return old;
        }

        //取反获取需要插入的位置
        index = ~index;
        //如果数组已经满了
        if (osize >= mHashes.length) {
            final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                    : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

            if (DEBUG) Log.d(TAG, "put: grow from " + mHashes.length + " to " + n);

            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
            //获取缓存，或者创建新的数组
            allocArrays(n);

            if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                throw new ConcurrentModificationException();
            }

            //进行复制
            if (mHashes.length > 0) {
                if (DEBUG) Log.d(TAG, "put: copy 0-" + osize + " to 0");
                System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
                System.arraycopy(oarray, 0, mArray, 0, oarray.length);
            }

            //尝试添加到缓存
            freeArrays(ohashes, oarray, osize);
        }

         //如果在数组内部进行插入
        if (index < osize) {
            if (DEBUG) Log.d(TAG, "put: move " + index + "-" + (osize-index)
                    + " to " + (index+1));
            System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
            System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
        }

        if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
            if (osize != mSize || index >= mHashes.length) {
                throw new ConcurrentModificationException();
            }
        }
        //最后赋值
        mHashes[index] = hash;
        mArray[index<<1] = key;
        mArray[(index<<1)+1] = value;
        mSize++;
        return null;
    }
```

通过 hash 获取对应下标的过程和 之前的有点不同，因为 hashCode 的不唯一性

```text
int indexOf(Object key, int hash) {
        final int N = mSize;

        // Important fast case: if nothing is in here, nothing to look for.
        if (N == 0) {
            return ~0;
        }

        //先使用二分进行查找
        int index = binarySearchHashes(mHashes, N, hash);

        // If the hash code wasn't found, then we have no entry for this key.
        if (index < 0) {
            return index;
        }
         //如果查找成功，并且对应的 key 是相同的，即 hashCode 相同且 equals 返回 true 
        // If the key at the returned index matches, that's what we want.
        if (key.equals(mArray[index<<1])) {
            return index;
        }

        //前面的 equals 返回 false 则继续查找
        //先从index 往 数组后面找 

        // Search for a matching key after the index.
        int end;
        for (end = index + 1; end < N && mHashes[end] == hash; end++) {
            if (key.equals(mArray[end << 1])) return end;
        }

        //再从 index 往数组前面找
        // Search for a matching key before the index.
        for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
            if (key.equals(mArray[i << 1])) return i;
        }

        // Key not found -- return negative value indicating where a
        // new entry for this key should go.  We use the end of the
        // hash chain to reduce the number of array entries that will
        // need to be copied when inserting.
        //查找失败则返回插入的位置 ，相同的 hashCode 
        //插入是插在后面。
        return ~end;
    }
```

对于get 方法就相对简单，这里就不做分析。

## 与 HashMap 的对比

优点 - 在**数据量少时，内存利用率高**，不用创建新的实体 Entry - 迭代效率高，使用数组下标，相比于HashMap迭代使用迭代器，要快 缺点 - 存取因 二分查找的 O(log n ) 复杂度高，效率低 - ArrayMap 没有实现 Serializable，不利于在 Android 中借助 Bundle 传输。

因此 适用场景元素数量较少但是查询多，插入数据和删除数据不频繁的情况。



# 参考
https://zhuanlan.zhihu.com/p/152230430

https://juejin.cn/post/6844903442901630983#heading-15