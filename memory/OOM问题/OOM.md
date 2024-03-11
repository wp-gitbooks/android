![image-20210510143229084](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210510143516.png)





![image-20210510143257024](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210510143257.png)



![image-20210510143308625](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210510143308.png)







# 线索



现象是什么？（what）

常见的OOM？怎么规避OOM

原因？





怎么定位？怎么解决？



# 参考

https://xiaolong.li/2019/09/27/OOM-error/



https://my.oschina.net/u/4416039/blog/3308786



# 概述

## 什么是OOM？

就是我们常见的： **java.lang.OutOfMemoryError** ，在应用开发中，是比较常见的一种异常，主要分为三种：

> OutOfMemoryError： PermGen space
> OutOfMemoryError： Java heap space
> OutOfMemoryError： unable to create new native thread



# 内存泄露 和 内存溢出 的区别

## 内存泄露[Memory Leak]

程序在申请内存后，**无法释放已申请的内存空间**，一次内存泄漏似乎不会有大的影响，但内存泄漏堆积后的后果就是内存溢出。

## 内存溢出[Out Of Memory]

程序申请内存时，**没有足够的内存供申请者使用**，就是内存不够用，此时就会报错OOM，即所谓的内存溢出。

## 两者关系

> 内存泄漏的堆积最终会导致内存溢出

## 内存泄漏的分类

> 常发性内存泄漏

发生内存泄漏的代码会被多次执行到，每次被执行的时候都会导致一块内存泄漏。

> 偶发性内存泄漏

发生内存泄漏的代码只有在某些特定环境或操作过程下才会发生。常发性和偶发性是相对的。对于特定的环境，偶发性的也许就变成了常发性的。所以测试环境和测试方法对检测内存泄漏至关重要。

> 一次性内存泄漏

发生内存泄漏的代码只会被执行一次，或者由于算法上的缺陷，导致总会有一块仅且一块内存发生泄漏。比如，在类的构造函数中分配内存，在析构函数中却没有释放该内存，所以内存泄漏只会发生一次。

> 隐式内存泄漏

程序在运行过程中不停的分配内存，但是直到结束的时候才释放内存。严格的说这里并没有发生内存泄漏，因为最终程序释放了所有申请的内存。但是对于一个服务器程序，需要运行几天，几周甚至几个月，不及时释放内存也可能导致最终耗尽系统的所有内存。所以，称这类内存泄漏为隐式内存泄漏。



# OOM的危害

> 应用服务异常
> 线程异常
> 程序崩溃
> 其他未知问题



# 怎么解决

既然OOM这么恐怖，那么我们应该如何排查定位，并结局问题呢？

## 定位进程

### 模拟内存溢出

编写一段OutOfMemoryError测试代码，用来模拟出 OOM 场景。

```
/**
 * @Author lixiaolong
 * created by xlli5 on 2019/9/28 9:47 PM use IntelliJ IDEA
 */
@Service(ServiceNames.TEST_MEMORY + ServiceVersion.V_1)
public class MemoryTestService extends ServiceHandler<TestMemoryRequest, TestMemoryResponse> {

    @Override
    protected void validateReqParam(TestMemoryRequest req, TestMemoryResponse resp) {

    }

    @Override
    public void process(LogInfo logInfo, TestMemoryRequest req, TestMemoryResponse resp) {
        List<OOMObject> list = new ArrayList<OOMObject>();
        int count = 100000;
        while (true) {
            list.add(new OOMObject());

        }
    }

    static class OOMObject {

    }
}
```

给程序分配内存

```
$JAVA_OPTS -Xms200m -Xmx200m
```

[![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210525103722.png)](https://xiaolong.li/images/exception/2019-09-28-oom-startup.png)

程序启动后，并发调用一段时间后出现OOM现象。

[![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210525103728.png)](https://xiaolong.li/images/exception/2019-09-28-oom-error.png)

### dump堆栈信息

获取进程的PID并使用jvm自带的jmap命令dump进程的堆栈信息，如下：

```
ps -ef | grep xiaolong-test-service

jmap -dump:format=b,file=heap.hprof 6196
```

[![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210525103732.png)](https://xiaolong.li/images/exception/2019-09-28-oom-jmap-dump.png)

这里可以在程序中配置启动参数，在内存溢出时，自动输出dump文件。

```
VM options:

-XX:+PrintGCDetails
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/xiaolong-test-service/dump/
```

### 使用MAT分析堆栈信息

启动MAT打开dump的文件heap.hprof进行分析。

[![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210525103736.png)](https://xiaolong.li/images/exception/2019-09-28-oom-mat-index.png)

Unreachable指的是可以被垃圾回收器回收的对象，但是由于没有GC发生，所以没有释放，这时抓的内存使用中的Unreachable就是这些对象。

![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210525104000.png)

> Shallow Heap（浅堆） 表示该对象自身占用的堆内存，不包括它引用的对象。
> 针对非数组类型的对象，它的大小就是对象与它所有的成员变量大小的总和。

> Retained Heap（深堆） 表示当前对象大小+当前对象可直接或间接引用到的对象的大小总和。
> 换句话说，Retained Size就是当前对象被GC后，从Heap上总共能释放掉的内存。
> 不过，释放的时候还要排除被GC Roots直接或间接引用的对象。他们暂时不会被被当做Garbage。

还能看到一些堆栈分析概述，系统信息，以及类直方图。

![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210525104114.png)

[![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210525103750.png)](https://xiaolong.li/images/exception/2019-09-28-oom-mat-system-properties.png)

[![image](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210525103755.png)](https://xiaolong.li/images/exception/2019-09-28-oom-mat-class-histogram.png)

根据上图可知，OOMObject对象可能是造成内存泄漏的原因。



## 工具

### 使用dmesg命令查看系统日志

dmesg |grep -E ‘kill|oom|out of memory’，可以查看操作系统启动后的系统日志，这里就是查看跟内存溢出相关联的系统日志。

