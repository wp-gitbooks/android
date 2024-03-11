# 问题

Major GC和Full GC触发条件区别



# 1: Java Minor GC、Major GC和Full GC之间的区别

- Minor GC
  - Minor GC指新生代GC，即发生在新生代（包括Eden区和Survivor区）的垃圾回收操作，当新生代无法为新生对象分配内存空间的时候，会触发Minor GC。因为新生代中大多数对象的生命周期都很短，所以发生Minor GC的频率很高，虽然它会触发stop-the-world，但是它的回收速度很快。
- Major GC
  - Major GC清理Tenured区，用于回收老年代，出现Major GC通常会出现至少一次Minor GC。
- Full GC
  - Full GC是针对整个新生代、老生代、元空间（metaspace，java8以上版本取代perm gen）的全局范围的GC。Full GC不等于Major GC，也不等于Minor GC+Major GC，发生Full GC需要看使用了什么垃圾收集器组合，才能解释是什么样的垃圾回收。

# 2:MinorGC触发条件

**虚拟机在进行minorGC之前会判断****老年代****最大的可用连续空间是否大于新生代的所有对象总空间**

  ***\*1、如果大于的话，直接执行minorGC\****

  **2、如果小于，判断是否开启HandlerPromotionFailure，没有开启直接FullGC**

  **3、如果开启了HanlerPromotionFailure, JVM会判断老年代的最大连续内存空间是否大于历次晋升（****晋级老年代对象的平均大小）平均值的大小****，如果小于直接执行FullGC**

  **4、如果大于的话，执行minorGC**

对于HandlerPromotionFailure，我们可以这样理解，在发生Minor GC之前，虚拟机会先检查老年代的最大的连续内存空间是否大于新生代的所有对象的空间，如果这个条件成立，Minor GC是安全的。如果不成立虚拟机会查看HanlerPromotionFailure 设置值是否允许担当失败，如果允许，那么会继续检查老年代最大可用的连续内存空间是否大于历次晋级到老年代对象的平均大小，如果大于就尝试一次Minor GC， 如果小于，或者HanlerPromotionFailure 不愿承担风险就要进行一次Full GC 。

上面的担保就好比你去买房，要去银行贷款。如果你的贷款金额比较大，那么一般银行会需要你提供担保人。说白了，银行就是担心你每个月的工资不够还贷款。这里的老年代就相当于你的担保人。如果你的担保人的每个月的收入都不够你的月供。那银行肯定不会贷款给你的。在jvm中就是肯定会直接执行Full GC了。(强行比喻一番，能理解就好。。)

# 3:FullGC触发条件

- **老年代空间不足**

   如果创建一个大对象，Eden区域当中放不下这个大对象，会直接保存在老年代当中，如果老年代空间也不足，就会触发Full GC。为了避免这种情况，最好就是不要创建太大的对象。

- 持久代空间不足

   如果有持久代空间的话，系统当中需要加载的类，调用的方法很多，同时持久代当中没有足够的空间，就出触发一次Full GC

- YGC出现promotion failure

  promotion failure发生在Young GC, 如果Survivor区当中存活对象的年龄达到了设定值，会就将Survivor区当中的对象拷贝到老年代，如果老年代的空间不足，就会发生promotion failure， 接下去就会发生Full GC.

- 统计YGC发生时晋升到老年代的平均总大小大于老年代的空闲空间

   在发生YGC是会判断，是否安全，这里的安全指的是，当前老年代空间可以容纳YGC晋升的对象的平均大小，如果不安全，就不会执行YGC,转而执行FullGC。

- 显示调用System.gc

这里调用了 System.gc 并不一定会立马就触发FullGC





# 参考

https://blog.csdn.net/luzhensmart/article/details/103782340