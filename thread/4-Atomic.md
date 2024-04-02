# 概述
[[JUC原子类-CAS-Unsafe和原子类详解]]

Java的`java.util.concurrent`包除了提供底层锁、并发集合外，还提供了一组原子操作的封装类，它们位于`java.util.concurrent.atomic`包

在Atomic包里一共有12个类，四种原子更新方式，分别是原子更新基本类型，原子更新数组，原子更新引用和原子更新字段。Atomic包里的类基本都是使用Unsafe实现的包装类

- 原子更新基本类型类： AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference
- 原子更新数组类：AtomicIntegerArray，AtomicLongArray
- 原子更新引用类型：AtomicMarkableReference，AtomicStampedReference，AtomicReferenceArray
- 原子更新字段类：AtomicLongFieldUpdater，AtomicIntegerFieldUpdater，AtomicReferenceFieldUpdater



# AtomicInteger

- 增加值并返回新值：`int addAndGet(int delta)`
- 加1后返回新值：`int incrementAndGet()`
- 获取当前值：`int get()`
- 用CAS方式设置：`int compareAndSet(int expect, int update)`



## 原理

Atomic类是通过无锁（lock-free）的方式实现的线程安全（thread-safe）访问。它的主要原理是利用了CAS：Compare and Set。



1、利用到CAS机制

2、CAS（cpu层面对比较修改加锁）



## Compare And Swap

CAS 的含义是“我认为原有的值应该是什么，如果是，则将原有的值更新为新值，否则不做修改，并告诉我原来的值是多少”。（这段描述引自《Java并发编程实践》）
    简单的来说，CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。**当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则返回V**。这是一种乐观锁的思路，它相信在它修改之前，没有其它线程去修改它；而Synchronized是一种悲观锁，它认为在它修改之前，一定会有其它线程去修改它，悲观锁效率很低

### volatile变量
```
private volatile int value;  
```

​    首先声明了一个volatile变量value，我们知道volatile保证了变量的内存可见性，也就是所有工作线程中同一时刻都可以得到一致的值。

```
public final int get() {  
    return value;  
}  
```

### Compare And Set

```
// setup to use Unsafe.compareAndSwapInt for updates  
private static final Unsafe unsafe = Unsafe.getUnsafe();  
private static final long valueOffset;// 注意是静态的  
  
static {  
  try {  
    valueOffset = unsafe.objectFieldOffset  
        (AtomicInteger.class.getDeclaredField("value"));// 反射出value属性，获取其在内存中的位置  
  } catch (Exception ex) { throw new Error(ex); }  
}  
  
public final boolean compareAndSet(int expect, int update) {  
  return unsafe.compareAndSwapInt(this, valueOffset, expect, update);  
}  
```

​    比较并设置，这里利用Unsafe类的JNI方法实现，使用CAS指令，可以保证读-改-写是一个原子操作。compareAndSwapInt有4个参数，this - 当前AtomicInteger对象，Offset - value属性在内存中的位置(需要强调的是不是value值在内存中的位置)，expect - 预期值，update - 新值，根据上面的CAS操作过程，当内存中的value值等于expect值时，则将内存中的value值更新为update值，并返回true，否则返回false。在这里我们有必要对Unsafe有一个简单点的认识，从名字上来看，不安全，确实，这个类是用于执行低级别的、不安全操作的方法集合，这个类中的方法大部分是对内存的直接操作，所以不安全，但当我们使用反射、并发包时，都间接的用到了Unsafe。

### 循环设置

```
public final int incrementAndGet() {  
    for (;;) {// 这样优于while(true)  
        int current = get();// 获取当前值  
        int next = current + 1;// 设置更新值  
        if (compareAndSet(current, next))  
            return next;  
    }  
}  
```

​    循环内，获取当前值并设置更新值，调用compareAndSet进行CAS操作，如果成功就返回更新值，否则重试到成功为止。这里可能存在一个隐患，那就是循环时间过长，总是在当前线程compareAndSet时，有另一个线程设置了value(点子太背了)，这个当然是属于小概率时间，目前Java貌似还不能处理这种情况。



## CAS编写`incrementAndGet()`
var.get 是volatile，内存可见的

CAS在CPU层面加锁，保障比较和修改是原子的

```
public int incrementAndGet(AtomicInteger var) {
    int prev, next;
    do {
        prev = var.get();
        next = prev + 1;
    } while ( ! var.compareAndSet(prev, next));
    return next;
}
```



`incrementAndGet()`方法的实现：

```
/**
 * Atomically increments by one the current value.
 * @return the updated value
 */
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```


```
/**
 * Atomically adds the given value to the current value of a field
 * or array element within the given object <code>o</code>
 * at the given <code>offset</code>.
 *
 * @param o object/array to update the field/element in
 * @param offset field/element offset
 * @param delta the value to add
 * @return the previous value
 * @since 1.8
 */
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
}


public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);
```

这是一个循环，offset是变量v在内存中相对于对象o起始位置的偏移，传给JNI层用来计算这个value的内存绝对地址。

然后找到JNI的实现代码，来看 native层的`compareAndSwapInt()`方法的实现。这个方法的实现是这样的：

```
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);    //计算变量的内存绝对地址
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```



这个函数其实很简单，就是去看一下obj 的 offset 上的那个位置上的值是多少，如果是 e，那就把它更新为 x，返回true，如果不是 e，那就什么也不做，并且返回false。里面的核心方法是`Atomic::compxchg()`，这个方法所属的类文件是在os_cpu目录下面，由此可以看出这个类是和CPU操作有关，进入代码如下：

```
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}
```

这个方法里面都是汇编指令，看到`LOCK_IF_MP`也有锁指令实现的原子操作，其实**CAS也算是有锁**操作，只不过是由CPU来触发，比synchronized性能好的多。


## 用途
使用`java.util.concurrent.atomic`提供的原子操作可以简化多线程编程：
- 原子操作实现了无锁的线程安全；
- 适用于计数器，累加器等。

# 原子更新数组类



# 原子更新引用类型



# 原子更新字段类





# 实现自旋锁

```
/**
 * 使用AtomicInteger实现自旋锁
 */
public class SpinLock {

	private AtomicInteger state = new AtomicInteger(0);

	/**
	 * 自旋等待直到获得许可
	 */
	public void lock(){
		for (;;){
			//CAS指令要锁总线，效率很差。所以我们通过一个if判断避免了多次使用CAS指令。
			if (state.get() == 1) {
				continue;
			} else if(state.compareAndSet(0, 1)){
				return;
			}
		}
	}

	public void unlock(){
		state.set(0);
	}
}
```



# 实现可等待的锁

```
/**
 * 使用AtomicInteger实现可等待锁
 */
public class BlockLock implements Lock {
    
    private AtomicInteger state = new AtomicInteger(0);
    private ConcurrentLinkedQueue<Thread> waiters = new ConcurrentLinkedQueue<>();

    @Override
    public void lock() {
        if (state.compareAndSet(0, 1)) {
            return;
        }
        //放到等待队列
        waiters.add(Thread.currentThread());

        for (;;) {
            if (state.get() == 0) {
                if (state.compareAndSet(0, 1)) {
                    waiters.remove(Thread.currentThread());
                    return;
                }
            } else {
                LockSupport.park();     //挂起线程
            }
        }
    }

    @Override
    public void unlock() {
        state.set(0);
        //唤醒等待队列的第一个线程
        Thread waiterHead = waiters.peek();
        if(waiterHead != null){
            LockSupport.unpark(waiterHead);     //唤醒线程
        }
    }
}
```