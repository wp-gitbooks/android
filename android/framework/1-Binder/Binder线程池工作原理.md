# 参考

http://gityuan.com/2016/10/29/binder-thread-pool/



https://www.daimajiaoliu.com/daima/4871b7de59003e4



https://segmentfault.com/a/1190000020501252

https://www.jianshu.com/p/ea4fc6aefaa8





# 概述

```
frameworks/base/cmds/app_process/app_main.cpp
frameworks/native/libs/binder/ProcessState.cpp
framework/native/libs/binder/IPCThreadState.cpp
kernel/drivers/staging/android/binder.c
```



![image-20210512141713128](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210512141713.png)



# 总结



![binder_thread_create](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210404224753.jpg)

# 面试题

## binder通信的内存大小限制。（1M和128k）？

### 结论

#### 1.通过手写open，mmap初始化Binder服务的限制是4MB

#### 2.通过ProcessState初始化Binder服务的限制是1MB-8KB





由Zygote孵化而来的用户进程，所映射的Binder内存大小是不到1M的，准确说是 1* 1024 * 1024 - (4096 *2) ;

```
#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))
```

1MB-4KB*2





ServiceManager进程：    bs = binder_open(128*1024); 

有个特殊的进程ServiceManager进程，它为自己申请的Binder内核空间是128K，这个同ServiceManager的用途是分不开的，ServcieManager主要面向系统Service，只是简单的提供一些addServcie，getService的功能，不涉及多大的数据传输，因此不需要申请多大的内存：

```
int main(int argc, char **argv)
{
    struct binder_state *bs;
    void *svcmgr = BINDER_SERVICE_MANAGER;

        // 仅仅申请了128k
    bs = binder_open(128*1024);
 if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    svcmgr_handle = svcmgr;
    binder_loop(bs, svcmgr_handler);
    return 0;
}    
```



而在内核中，其实也有个限制，是4M，不过由于APP中已经限制了不到1M，这里的限制似乎也没多大用途：

```
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;
    struct vm_struct *area;
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;
    struct binder_buffer *buffer;
    //限制不能超过4M
    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;
    。。。
    }复制代码
```



### 能否不用ProcessState来初始化Binder服务，来突破1M-8KB的限制？

答案是当然可以了，Binder服务的初始化有两步，open打开Binder驱动，mmap在Binder驱动中申请内核空间内存，所以我们只要手写open，mmap就可以轻松突破这个限制。源码中已经给了类似的例子。

[frameworks](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2F)/[native](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fnative%2F)/[cmds](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fnative%2Fcmds%2F)/[servicemanager](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fnative%2Fcmds%2Fservicemanager%2F)/[bctest.c](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fnative%2Fcmds%2Fservicemanager%2Fbctest.c)



```cpp
int main(int argc, char **argv)
{
    struct binder_state *bs;
    uint32_t svcmgr = BINDER_SERVICE_MANAGER;
    uint32_t handle;

    bs = binder_open("/dev/binder", 128*1024);//我们可以把这个数值改成2*1024*1024就可以突破这个限制了
    if (!bs) {
        fprintf(stderr, "failed to open binder driver\n");
        return -1;
    }
```

[frameworks](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2F)/[native](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fnative%2F)/[cmds](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fnative%2Fcmds%2F)/[servicemanager](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fnative%2Fcmds%2Fservicemanager%2F)/[binder.c](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fnative%2Fcmds%2Fservicemanager%2Fbinder.c)



```rust
struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    ...//省略部分代码
    bs->fd = open(driver, O_RDWR | O_CLOEXEC);
    ....//省略部分代码
    bs->mapsize = mapsize;//这里mapsize=128*1024
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    ....//省略部分代码
}
```



### 难道Binder驱动不怕我们传递一个超级大的数字进去吗？

其实是我们想多了，在Binder驱动中mmap的具体实现中还有一个4M的限制
 /[drivers](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2Fkernel_3.18%2Fxref%2Fdrivers%2F)/[staging](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2Fkernel_3.18%2Fxref%2Fdrivers%2Fstaging%2F)/[android](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2Fkernel_3.18%2Fxref%2Fdrivers%2Fstaging%2Fandroid%2F)/[binder.c](https://links.jianshu.com/go?to=http%3A%2F%2Fandroidxref.com%2Fkernel_3.18%2Fxref%2Fdrivers%2Fstaging%2Fandroid%2Fbinder.c)



```rust
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;
    struct vm_struct *area;
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;
    struct binder_buffer *buffer;

    if (proc->tsk != current)
        return -EINVAL;

    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;//如果申请的size大于4MB了，会在驱动中被修改成4MB

    binder_debug(BINDER_DEBUG_OPEN_CLOSE,
             "binder_mmap: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
             proc->pid, vma->vm_start, vma->vm_end,
             (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
             (unsigned long)pgprot_val(vma->vm_page_prot));
```



### 问题：一次Binder通信最大可以传输多大的数据？

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210512143052)



### 再问一下自己，自己写的APP能否突破1M-8KB的限制

答案是理论上可以，但是不建议这样子操作，因为Binder驱动中并没有对open，mmap有调用次数的限制，App可以通过JNI调用open，mmap来突破这个限制，但是会对当前正在进行Binder调用的APP造成不可想象问题，当然可以先close Binder驱动。但是一旦这个APP没有Binder通信了，这个APP就不能正常使用了，APP和其他应用，AMS，WMS的交互可都是依赖于Binder通信，所以还是那句话，无Binder无Android







## binder线程池默认最大数量（15）？

每个进程的binder线程池的线程个数上限为15，只有第一个Binder主线程(也就是Binder_1线程)是由应用程序主动创建，Binder线程池的普通线程都是由Binder驱动根据IPC通信需求创建



## Binder线程池的工作过程是什么样？

