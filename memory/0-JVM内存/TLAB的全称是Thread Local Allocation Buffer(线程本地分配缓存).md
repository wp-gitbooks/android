TLAB的全称是Thread Local Allocation Buffer，翻译过来就是线程本地分配缓存。

# TLAB是线程专用的内存分配区域，可以解决内存分配冲突问题。

在线程初始化时，同时也会申请一块指定大小的内存，只给当前线程使用，这样每个线程都单独拥有一个空间，如果需要分配内存，就在自己的空间上分配，这样就不存在竞争的情况，可以大大提升分配效率 TLAB空间的内存非常小，缺省情况下仅占有整个Eden空间的1%，也可以通过选项-XX:TLABWasteTargetPercent设置TLAB空间所占用Eden空间的百分比大小。

# TLAB的本质

其实是三个指针管理的区域：`start`，`top` 和 `end`，每个线程都会从Eden分配一块空间，例如说100KB，作为自己的TLAB，其中 start 和 end 是占位用的，标识出 eden 里被这个 TLAB 所管理的区域，卡住eden里的一块空间不让其它线程来这里分配。TLAB只是让每个线程有私有的分配指针，但底下存对象的内存空间还是给所有线程访问的，只是其它线程无法在这个区域分配而已。 从这一点看，它被翻译为 线程私有分配区更为合理一点 当一个TLAB用满（分配指针top撞上分配极限end了），就新申请一个TLAB，而在老TLAB里的对象还留在原地什么都不用管——它们无法感知自己是否是曾经从TLAB分配出来的，而只关心自己是在eden里分配的。由于TALB空间本身较小（默认只占Eden区1%），所以就很容易出现TALB剩余区域不足以存储新对象的情况，这时线程会把新对象存到新申请的TALB中，这样原有的TALB中剩余区域就会被浪费，造成内存泄漏。那么如何解决内存泄漏呢？

# 最大浪费空间

由于TALB内存浪费现象较为严重，所以JVM开发人员提出了一个最大浪费空间对TALB进行约束。 当TALB剩余空间存不下新对象时，会进行一个判断：

-   ① 如果当前TALB剩余空间小于最大浪费空间，则TALB所属线程会向JVM申请一个新的TALB区域存储新对象，如果依旧存储不下，则对象会放在Eden区创建。
-   ② 如果当前TALB剩余空间大于最大浪费空间，则对象直接去Eden区创建

# TLAB的局限性

虽然TALB解决了指针碰撞在多线程场景下的问题，并且通过最大浪费空间可以减少内存泄漏，但其本身依旧有一些缺点： ① GC更频繁： 由于每个TALB所占用的空间都要比线程实际需要的空间大小大一些（因为不可能每个TALB都刚好存满，也就是TALB空间浪费更严重），所以一批对象直接存储在Eden区会比存储在TALB区占用更多的空间，进而容易引发Minor GC。 ② TALB允许内存浪费，会导致Eden区内存不连续

# 拓展

## 创建对象的内存分配   [[0-垃圾回收-内存分配#7 内存分配与回收策略]]

![146c7be92a8d4ae3840e407d5bed9073_tplv-k3u1fbpfcp-watermark.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304061119601.webp)

1.  编译器通过逃逸分析判断对象是在栈上分配还是堆上分配，如果是堆上分配则进入下一步。（开启逃逸分析需要设置jvm参数）
2.  如果tlab可以放下该对象则在tlab上分配，否则进入下一步。
3.  重新申请一个tlab，再尝试存放该对象，如果放不下则进入下一步。
4.  在eden区加锁，尝试在eden区存放，若存放不下则进入下一步。
5.  执行一次Young GC。
6.  Young GC后若eden区仍放不下该对象，则直接在老年代分配。

# 测试

测试代码

```java
/**
 * 测试TLAB
 */
public class TLABTest {
    public static void get() {
        byte[] bytes = new byte[2];
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 10000000; i++) {
            get();
        }
        long end = System.currentTimeMillis();
        System.out.println(end - start);
    }
}
复制代码
```

先测试禁用TLAB的情况：  
调整虚拟机参数如下：

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304061119602.webp)

-   -server：以server模式运行，支持逃逸分析参数DoEscapeAnalysis
-   -XX:-DoEscapeAnalysis：关闭逃逸分析，避免出现栈上分配影响效果
-   -XX:-BackgroundCompilation：禁止后台编译
-   -XX:-UseTLAB：禁用TLAB 运行结果如下：

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304061119603.webp) 再测试启用TLAB的情况：  
调整虚拟机参数如下：再测试启用TLAB的情况： ![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304061119604.webp) -XX:+UseTLAB：启用TLAB  
运行结果如下：

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304061119605.webp) 对比两个时间，可以发现TLAB是否启用确实影响着对象分配的效率。

  

作者：丶西红柿丶  
链接：https://juejin.cn/post/7107292213758918670  
来源：稀土掘金  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。