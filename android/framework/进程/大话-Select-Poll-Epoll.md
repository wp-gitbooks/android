提到select、poll、epoll相信大家都耳熟能详了，三个都是IO多路复用的机制，可以监视多个描述符的读/写等事件，一旦某个描述符就绪（一般是读或者写事件发生了），就能够将发生的事件通知给关心的应用程序去处理该事件。本质上，**select、poll、epoll本质上都是同步I/O**，相信大家都读过Richard Stevens的经典书籍UNP（UNIX:registered: Network Programming），书中给出了5种IO模型：

* 1、blocking IO - 阻塞IO 
* 2、nonblocking IO - 非阻塞IO 
* 3、IO multiplexing - IO多路复用 
* 4、signal driven IO - 信号驱动IO 
* 5、asynchronous IO - 异步IO

其中前面4种IO都可以归类为synchronous IO - 同步IO，在介绍select、poll、epoll之前，首先介绍一下这几种IO模型，signal driven IO平时用的比较少，这里就不介绍了。

## 1 IO - 同步、异步、阻塞、非阻塞

下面以network IO中的read读操作为切入点，来讲述同步（synchronous） IO和异步（asynchronous） IO、阻塞（blocking） IO和非阻塞（non-blocking）IO的异同。一般情况下，一次网络IO读操作会涉及两个系统对象：(1) 用户进程(线程)Process；(2)内核对象kernel，两个处理阶段：

```javascript
[1] Waiting for the data to be ready - 等待数据准备好
[2] Copying the data from the kernel to the process - 将数据从内核空间的buffer拷贝到用户空间进程的buffer
```

IO模型的异同点就是区分在这两个系统对象、两个处理阶段的不同上。

### 1.1 同步IO 之 Blocking IO

![img](https://blog-10039692.file.myqcloud.com/1500016932293_3417_1500016932577.png)

如上图所示，用户进程process在Blocking IO读recvfrom操作的两个阶段都是等待的。在数据没准备好的时候，process原地等待kernel准备数据。kernel准备好数据后，process继续等待kernel将数据copy到自己的buffer。在kernel完成数据的copy后process才会从recvfrom系统调用中返回。

### 1.2 同步IO 之 NonBlocking IO

![img](https://blog-10039692.file.myqcloud.com/1500017008242_1297_1500017008596.png)

从图中可以看出，process在NonBlocking IO读recvfrom操作的第一个阶段是不会block等待的，如果kernel数据还没准备好，那么recvfrom会立刻返回一个EWOULDBLOCK错误。当kernel准备好数据后，进入处理的第二阶段的时候，process会等待kernel将数据copy到自己的buffer，在kernel完成数据的copy后process才会从recvfrom系统调用中返回。

### 1.3 同步IO 之 IO multiplexing

![img](https://blog-10039692.file.myqcloud.com/1500017024989_2800_1500017025229.png)

IO多路复用，就是我们熟知的select、poll、epoll模型。从图上可见，在IO多路复用的时候，process在两个处理阶段都是block住等待的。初看好像IO多路复用没什么用，其实select、poll、epoll的优势在于可以以较少的代价来同时监听处理多个IO。

### 1.4 异步IO

![img](https://blog-10039692.file.myqcloud.com/1500017078734_6117_1500017078936.png)

从上图看出，异步IO要求process在recvfrom操作的两个处理阶段上都不能等待，也就是process调用recvfrom后立刻返回，kernel自行去准备好数据并将数据从kernel的buffer中copy到process的buffer在通知process读操作完成了，然后process在去处理。遗憾的是，linux的网络IO中是不存在异步IO的，linux的网络IO处理的第二阶段总是阻塞等待数据copy完成的。真正意义上的网络异步IO是Windows下的IOCP（IO完成端口）模型。

![img](https://blog-10039692.file.myqcloud.com/1500017105443_4641_1500017105783.png)

很多时候，我们比较容易混淆non-blocking IO和asynchronous IO，认为是一样的。但是通过上图，几种IO模型的比较，会发现non-blocking IO和asynchronous IO的区别还是很明显的，non-blocking IO仅仅要求处理的第一阶段不block即可，而asynchronous IO要求两个阶段都不能block住。

## 2 Linux的socket 事件wakeup callback机制

言归正传，在介绍select、poll、epoll前，有必要说说linux(2.6+)内核的事件wakeup callback机制，这是IO多路复用机制存在的本质。Linux通过socket睡眠队列来管理所有等待socket的某个事件的process，同时通过wakeup机制来异步唤醒整个睡眠队列上等待事件的process，通知process相关事件发生。通常情况，socket的事件发生的时候，其会顺序遍历socket睡眠队列上的每个process节点，调用每个process节点挂载的callback函数。在遍历的过程中，如果遇到某个节点是排他的，那么就终止遍历，总体上会涉及两大逻辑：（1）睡眠等待逻辑；（2）唤醒逻辑。

（1）睡眠等待逻辑：涉及select、poll、epoll_wait的阻塞等待逻辑

```javascript
[1]select、poll、epoll_wait陷入内核，判断监控的socket是否有关心的事件发生了，如果没，则为当前process构建一个wait_entry节点，然后插入到监控socket的sleep_list
[2]进入循环的schedule直到关心的事件发生了
[3]关心的事件发生后，将当前process的wait_entry节点从socket的sleep_list中删除。
```

（2）唤醒逻辑：

```javascript
[1]socket的事件发生了，然后socket顺序遍历其睡眠队列，依次调用每个wait_entry节点的callback函数
[2]直到完成队列的遍历或遇到某个wait_entry节点是排他的才停止。
[3]一般情况下callback包含两个逻辑：1.wait_entry自定义的私有逻辑；2.唤醒的公共逻辑，主要用于将该wait_entry的process放入CPU的就绪队列，让CPU随后可以调度其执行。
```

下面就上面的两大逻辑，分别阐述select、poll、epoll的异同，为什么epoll能够比select、poll高效。

## 3 大话Select—1024

在一个高性能的网络服务上，大多情况下一个服务进程(线程)process需要同时处理多个socket，我们需要公平对待所有socket，对于read而言，那个socket有数据可读，process就去读取该socket的数据来处理。于是对于read，一个朴素的需求就是关心的N个socket是否有数据”可读”，也就是我们期待”可读”事件的通知，而不是盲目地对每个socket调用recv/recvfrom来尝试接收数据。我们应该block在等待事件的发生上，这个事件简单点就是”关心的N个socket中一个或多个socket有数据可读了”，当block解除的时候，就意味着，我们一定可以找到一个或多个socket上有可读的数据。另一方面，根据上面的socket wakeup callback机制，我们不知道什么时候，哪个socket会有读事件发生，于是，process需要同时插入到这N个socket的sleep_list上等待任意一个socket可读事件发生而被唤醒，当时process被唤醒的时候，其callback里面应该有个逻辑去检查具体那些socket可读了。

于是，select的多路复用逻辑就清晰了，select为每个socket引入一个poll逻辑，该poll逻辑用于收集socket发生的事件，对于可读事件来说，简单伪码如下：

```javascript
poll()
{
    //其他逻辑
    if (recieve queque is not empty)
    {
        sk_event |= POLL_IN；
    }
   //其他逻辑
}
```

接下来就到select的逻辑了，下面是select的函数原型：5个参数，后面4个参数都是in/out类型(值可能会被修改返回)

```javascript
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

当用户process调用select的时候，select会将需要监控的readfds集合拷贝到内核空间（假设监控的仅仅是socket可读），然后遍历自己监控的socket sk，挨个调用sk的poll逻辑以便检查该sk是否有可读事件，遍历完所有的sk后，如果没有任何一个sk可读，那么select会调用schedule_timeout进入schedule循环，使得process进入睡眠。如果在timeout时间内某个sk上有数据可读了，或者等待timeout了，则调用select的process会被唤醒，接下来select就是遍历监控的sk集合，挨个收集可读事件并返回给用户了，相应的伪码如下：

```javascript
for (sk in readfds)
{
    sk_event.evt = sk.poll();
    sk_event.sk = sk;
    ret_event_for_process;
}
```

通过上面的select逻辑过程分析，相信大家都意识到，select存在两个问题：

```javascript
[1] 被监控的fds需要从用户空间拷贝到内核空间
    为了减少数据拷贝带来的性能损坏，内核对被监控的fds集合大小做了限制，并且这个是通过宏控制的，大小不可改变(限制为1024)。
[2] 被监控的fds集合中，只要有一个有数据可读，整个socket集合就会被遍历一次调用sk的poll函数收集可读事件
    由于当初的需求是朴素，仅仅关心是否有数据可读这样一个事件，当事件通知来的时候，由于数据的到来是异步的，我们不知道事件来的时候，有多少个被监控的socket有数据可读了，于是，只能挨个遍历每个socket来收集可读事件。
```

到这里，我们有三个问题需要解决：

（1）被监控的fds集合限制为1024，1024太小了，我们希望能够有个比较大的可监控fds集合 （2）fds集合需要从用户空间拷贝到内核空间的问题，我们希望不需要拷贝 （3）当被监控的fds中某些有数据可读的时候，我们希望通知更加精细一点，就是我们希望能够从通知中得到有可读事件的fds列表，而不是需要遍历整个fds来收集。

## 4 大话poll—鸡肋

select遗留的三个问题中，问题(1)是用法限制问题，问题(2)和(3)则是性能问题。poll和select非常相似，poll并没着手解决性能问题，poll只是解决了select的问题(1)fds集合大小1024限制问题。下面是poll的函数原型，poll改变了fds集合的描述方式，使用了pollfd结构而不是select的fd_set结构，使得poll支持的fds集合限制远大于select的1024。poll虽然解决了fds集合大小1024的限制问题，但是，它并没改变大量描述符数组被整体复制于用户态和内核态的地址空间之间，以及个别描述符就绪触发整体描述符集合的遍历的低效问题。poll随着监控的socket集合的增加性能线性下降，poll不适合用于大并发场景。

```javascript
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

## 5 大话epoll—终极武功

select遗留的三个问题，问题(1)是比较好解决，poll简单两三下就解决掉了，但是poll的解决有点鸡肋。要解决问题(2)和(3)似乎比较棘手，要怎么解决呢？我们知道，在计算机行业中，有两种解决问题的思想：

```javascript
[1] 计算机科学领域的任何问题, 都可以通过添加一个中间层来解决
[2] 变集中(中央)处理为分散(分布式)处理
```

下面，我们看看，epoll在解决select的遗留问题(2)和(3)的时候，怎么运用这两个思想的。

### 5.1 fds集合拷贝问题的解决

对于IO多路复用，有两件事是必须要做的(对于监控可读事件而言)：1. 准备好需要监控的fds集合；2. 探测并返回fds集合中哪些fd可读了。细看select或poll的函数原型，我们会发现，每次调用select或poll都在重复地准备(集中处理)整个需要监控的fds集合。然而对于频繁调用的select或poll而言，fds集合的变化频率要低得多，我们没必要每次都重新准备(集中处理)整个fds集合。

于是，epoll引入了epoll_ctl系统调用，将高频调用的epoll_wait和低频的epoll_ctl隔离开。同时，epoll_ctl通过(EPOLL_CTL_ADD、EPOLL_CTL_MOD、EPOLL_CTL_DEL)三个操作来分散对需要监控的fds集合的修改，做到了有变化才变更，将select或poll高频、大块内存拷贝(集中处理)变成epoll_ctl的低频、小块内存的拷贝(分散处理)，避免了大量的内存拷贝。同时，对于高频epoll_wait的可读就绪的fd集合返回的拷贝问题，epoll通过内核与用户空间mmap(内存映射)同一块内存来解决。mmap将用户空间的一块地址和内核空间的一块地址同时映射到相同的一块物理内存地址（不管是用户空间还是内核空间都是虚拟地址，最终要通过地址映射映射到物理地址），使得这块物理内存对内核和对用户均可见，减少用户态和内核态之间的数据交换。

另外，epoll通过epoll_ctl来对监控的fds集合来进行增、删、改，那么必须涉及到fd的快速查找问题，于是，一个低时间复杂度的增、删、改、查的数据结构来组织被监控的fds集合是必不可少的了。在linux 2.6.8之前的内核，epoll使用hash来组织fds集合，于是在创建epoll fd的时候，epoll需要初始化hash的大小。于是epoll_create(int size)有一个参数size，以便内核根据size的大小来分配hash的大小。在linux 2.6.8以后的内核中，epoll使用红黑树来组织监控的fds集合，于是epoll_create(int size)的参数size实际上已经没有意义了。

### 5.2 按需遍历就绪的fds集合

通过上面的socket的睡眠队列唤醒逻辑我们知道，socket唤醒睡眠在其睡眠队列的wait_entry(process)的时候会调用wait_entry的回调函数callback，并且，我们可以在callback中做任何事情。为了做到只遍历就绪的fd，我们需要有个地方来组织那些已经就绪的fd。为此，epoll引入了一个中间层，一个双向链表(ready_list)，一个单独的睡眠队列(single_epoll_wait_list)，并且，与select或poll不同的是，epoll的process不需要同时插入到多路复用的socket集合的所有睡眠队列中，相反process只是插入到中间层的epoll的单独睡眠队列中，process睡眠在epoll的单独队列上，等待事件的发生。同时，引入一个中间的wait_entry_sk，它与某个socket sk密切相关，wait_entry_sk睡眠在sk的睡眠队列上，其callback函数逻辑是将当前sk排入到epoll的ready_list中，并唤醒epoll的single_epoll_wait_list。而single_epoll_wait_list上睡眠的process的回调函数就明朗了：遍历ready_list上的所有sk，挨个调用sk的poll函数收集事件，然后唤醒process从epoll_wait返回。 于是，整个过来可以分为以下几个逻辑：

（1）epoll_ctl EPOLL_CTL_ADD逻辑

```javascript
[1] 构建睡眠实体wait_entry_sk，将当前socket sk关联给wait_entry_sk，并设置wait_entry_sk的回调函数为epoll_callback_sk
[2] 将wait_entry_sk排入当前socket sk的睡眠队列上
```

回调函数epoll_callback_sk的逻辑如下：

```javascript
[1] 将之前关联的sk排入epoll的ready_list
[2] 然后唤醒epoll的单独睡眠队列single_epoll_wait_list
```

（2）epoll_wait逻辑

```javascript
[1] 构建睡眠实体wait_entry_proc，将当前process关联给wait_entry_proc，并设置回调函数为epoll_callback_proc
[2] 判断epoll的ready_list是否为空，如果为空，则将wait_entry_proc排入epoll的single_epoll_wait_list中，随后进入schedule循环，这会导致调用epoll_wait的process睡眠。
[3] wait_entry_proc被事件唤醒或超时醒来，wait_entry_proc将被从single_epoll_wait_list移除掉，然后wait_entry_proc执行回调函数epoll_callback_proc
```

回调函数epoll_callback_proc的逻辑如下：

```javascript
[1] 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件，对于监控可读事件而已，ready_list上的每个sk都是有数据可读的，这里的遍历必要的(不同于select/poll的遍历，它不管有没数据可读都需要遍历一些来判断，这样就做了很多无用功。)
[2] 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process。
```

（3）epoll唤醒逻辑 整个epoll的协议栈唤醒逻辑如下(对于可读事件而言)：

```javascript
[1] 协议数据包到达网卡并被排入socket sk的接收队列
[2] 睡眠在sk的睡眠队列wait_entry被唤醒，wait_entry_sk的回调函数epoll_callback_sk被执行
[3] epoll_callback_sk将当前sk插入epoll的ready_list中
[4] 唤醒睡眠在epoll的单独睡眠队列single_epoll_wait_list的wait_entry，wait_entry_proc被唤醒执行回调函数epoll_callback_proc
[5] 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件
[6] 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process。
```

epoll巧妙的引入一个中间层解决了大量监控socket的无效遍历问题。细心的同学会发现，epoll在中间层上为每个监控的socket准备了一个单独的回调函数epoll_callback_sk，而对于select/poll，所有的socket都公用一个相同的回调函数。正是这个单独的回调epoll_callback_sk使得每个socket都能单独处理自身，当自己就绪的时候将自身socket挂入epoll的ready_list。同时，epoll引入了一个睡眠队列single_epoll_wait_list，分割了两类睡眠等待。process不再睡眠在所有的socket的睡眠队列上，而是睡眠在epoll的睡眠队列上，在等待”任意一个socket可读就绪”事件。而中间wait_entry_sk则代替process睡眠在具体的socket上，当socket就绪的时候，它就可以处理自身了。

### 5.3 ET(Edge Triggered 边沿触发) vs LT(Level Triggered 水平触发)

**5.3.1 ET vs LT - 概念**

说到Epoll就不能不说说Epoll事件的两种模式了，下面是两个模式的基本概念

- Edge Triggered (ET) 边沿触发

.socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件

.socket的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时触发读事件

仅在缓冲区状态变化时触发事件，比如数据缓冲去从无到有的时候(不可读-可读)

- Level Triggered (LT) 水平触发

.socket接收缓冲区不为空，有数据可读，则读事件一直触发

.socket发送缓冲区不满可以继续写入数据，则写事件一直触发

符合思维习惯，epoll_wait返回的事件就是socket的状态

通常情况下，大家都认为ET模式更为高效，实际上是不是呢？下面我们来说说两种模式的本质：

我们来回顾一下，5.2节（3）epoll唤醒逻辑 的第五个步骤

```javascript
[5] 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件
```

大家是不是有个疑问呢：挂在ready_list上的sk什么时候会被移除掉呢？其实，sk从ready_list移除的时机正是区分两种事件模式的本质。因为，通过上面的介绍，我们知道ready_list是否为空是epoll_wait是否返回的条件。于是，在两种事件模式下，步骤5如下：

对于Edge Triggered (ET) 边沿触发：

```javascript
[5] 遍历epoll的ready_list，将sk从ready_list中移除，然后调用该sk的poll逻辑收集发生的事件
```

对于Level Triggered (LT) 水平触发：

```javascript
[5.1] 遍历epoll的ready_list，将sk从ready_list中移除，然后调用该sk的poll逻辑收集发生的事件
[5.2] 如果该sk的poll函数返回了关心的事件(对于可读事件来说，就是POLL_IN事件)，那么该sk被重新加入到epoll的ready_list中。
```

对于可读事件而言，在ET模式下，如果某个socket有新的数据到达，那么该sk就会被排入epoll的ready_list，从而epoll_wait就一定能收到可读事件的通知(调用sk的poll逻辑一定能收集到可读事件)。于是，我们通常理解的缓冲区状态变化(从无到有)的理解是不准确的，准确的理解应该是是否有新的数据达到缓冲区。

而在LT模式下，某个sk被探测到有数据可读，那么该sk会被重新加入到read_list，那么在该sk的数据被全部取走前，下次调用epoll_wait就一定能够收到该sk的可读事件(调用sk的poll逻辑一定能收集到可读事件)，从而epoll_wait就能返回。

**5.3.2 ET vs LT - 性能**

通过上面的概念介绍，我们知道对于可读事件而言，LT比ET多了两个操作：(1)对ready_list的遍历的时候，对于收集到可读事件的sk会重新放入ready_list；(2)下次epoll_wait的时候会再次遍历上次重新放入的sk，如果sk本身没有数据可读了，那么这次遍历就变得多余了。 在服务端有海量活跃socket的时候，LT模式下，epoll_wait返回的时候，会有海量的socket sk重新放入ready_list。如果，用户在第一次epoll_wait返回的时候，将有数据的socket都处理掉了，那么下次epoll_wait的时候，上次epoll_wait重新入ready_list的sk被再次遍历就有点多余，这个时候LT确实会带来一些性能损失。然而，实际上会存在很多多余的遍历么？

先不说第一次epoll_wait返回的时候，用户进程能否都将有数据返回的socket处理掉。在用户处理的过程中，如果该socket有新的数据上来，那么协议栈发现sk已经在ready_list中了，那么就不需要再次放入ready_list，也就是在LT模式下，对该sk的再次遍历不是多余的，是有效的。同时，我们回归epoll高效的场景在于，服务器有海量socket，但是活跃socket较少的情况下才会体现出epoll的高效、高性能。因此，在实际的应用场合，绝大多数情况下，ET模式在性能上并不会比LT模式具有压倒性的优势，至少，目前还没有实际应用场合的测试表面ET比LT性能更好。

**5.3.3 ET vs LT - 复杂度**

我们知道，对于可读事件而言，在阻塞模式下，是无法识别队列空的事件的，并且，事件通知机制，仅仅是通知有数据，并不会通知有多少数据。于是，在阻塞模式下，在epoll_wait返回的时候，我们对某个socket_fd调用recv或read读取并返回了一些数据的时候，我们不能再次直接调用recv或read，因为，如果socket_fd已经无数据可读的时候，进程就会阻塞在该socket_fd的recv或read调用上，这样就影响了IO多路复用的逻辑(我们希望是阻塞在所有被监控socket的epoll_wait调用上，而不是单独某个socket_fd上)，造成其他socket饿死，即使有数据来了，也无法处理。

接下来，我们只能再次调用epoll_wait来探测一些socket_fd，看是否还有数据可读。在LT模式下，如果socket_fd还有数据可读，那么epoll_wait就一定能够返回，接着，我们就可以对该socket_fd调用recv或read读取数据。然而，在ET模式下，尽管socket_fd还是数据可读，但是如果没有新的数据上来，那么epoll_wait是不会通知可读事件的。这个时候，epoll_wait阻塞住了，这下子坑爹了，明明有数据你不处理，非要等新的数据来了在处理，那么我们就死扛咯，看谁先忍不住。

等等，在阻塞模式下，不是不能用ET的么？是的，正是因为有这样的缺点，ET强制需要在非阻塞模式下使用。在ET模式下，epoll_wait返回socket_fd有数据可读，我们必须要读完所有数据才能离开。因为，如果不读完，epoll不会在通知你了，虽然有新的数据到来的时候，会再次通知，但是我们并不知道新数据会不会来，以及什么时候会来。由于在阻塞模式下，我们是无法通过recv/read来探测空数据事件，于是，我们必须采用非阻塞模式，一直read直到EAGAIN。因此，ET要求socket_fd非阻塞也就不难理解了。

另外，epoll_wait原本的语意是：监控并探测socket是否有数据可读(对于读事件而言)。LT模式保留了其原本的语意，只要socket还有数据可读，它就能不断反馈，于是，我们想什么时候读取处理都可以，我们永远有再次poll的机会去探测是否有数据可以处理，这样带来了编程上的很大方便，不容易死锁造成某些socket饿死。相反，ET模式修改了epoll_wait原本的语意，变成了：监控并探测socket是否有新的数据可读。

于是，在epoll_wait返回socket_fd可读的时候，我们需要小心处理，要不然会造成死锁和socket饿死现象。典型如listen_fd返回可读的时候，我们需要不断的accept直到EAGAIN。假设同时有三个请求到达，epoll_wait返回listen_fd可读，这个时候，如果仅仅accept一次拿走一个请求去处理，那么就会留下两个请求，如果这个时候一直没有新的请求到达，那么再次调用epoll_wait是不会通知listen_fd可读的，于是epoll_wait只能睡眠到超时才返回，遗留下来的两个请求一直得不到处理，处于饿死状态。

**5.3.4 ET vs LT - 总结**

最后总结一下，ET和LT模式下epoll_wait返回的条件

- ET - 对于读操作

[1] 当接收缓冲buffer内待读数据增加的时候时候(由空变为不空的时候、或者有新的数据进入缓冲buffer)

[2] 调用epoll_ctl(EPOLL_CTL_MOD)来改变socket_fd的监控事件，也就是重新mod socket_fd的EPOLLIN事件，并且接收缓冲buffer内还有数据没读取。(这里不能是EPOLL_CTL_ADD的原因是，epoll不允许重复ADD的，除非先DEL了，再ADD) 因为epoll_ctl(ADD或MOD)会调用sk的poll逻辑来检查是否有关心的事件，如果有，就会将该sk加入到epoll的ready_list中，下次调用epoll_wait的时候，就会遍历到该sk，然后会重新收集到关心的事件返回。

- ET - 对于写操作

[1] 发送缓冲buffer内待发送的数据减少的时候(由满状态变为不满状态的时候、或者有部分数据被发出去的时候) [2] 调用epoll_ctl(EPOLL_CTL_MOD)来改变socket_fd的监控事件，也就是重新mod socket_fd的EPOLLOUT事件，并且发送缓冲buffer还没满的时候。

- LT - 对于读操作 LT就简单多了，唯一的条件就是，接收缓冲buffer内有可读数据的时候
- LT - 对于写操作 LT就简单多了，唯一的条件就是，发送缓冲buffer还没满的时候

在绝大多少情况下，ET模式并不会比LT模式更为高效，同时，ET模式带来了不好理解的语意，这样容易造成编程上面的复杂逻辑和坑点。因此，建议还是采用LT模式来编程更为舒爽。

## 参考资料

http://blog.chinaunix.net/uid-28541347-id-4238524.html http://blog.csdn.net/historyasamirror/article/details/5778378 http://blog.csdn.net/dog250/article/details/50528373 http://blog.csdn.net/zhangskd/article/details/16986931





# 参考

https://cloud.tencent.com/developer/article/1005481

