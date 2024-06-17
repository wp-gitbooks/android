epoll 于Linux 2.5.44引入，旨在替换select和poll系统函数。

相对于select和poll来说，epoll更加灵活高效:

- 没有监视描述符数量单进程1024限制
- epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

## epoll在Android中的应用

epoll机制在Android系统中扮演着一个很重要的角色，

1. 在MessageQueue的队列中，当队列为空时需要阻塞Looper线程，队列非空时候需要唤醒线程，使用到了epoll + eventfd(比pipe更高效的事件驱动，在[6.0引入](https://android.googlesource.com/platform/system/core/+/8892ce6^!/))机制，高效的管理着消息队列
2. java NIO中 Selector采用了epoll机制实现SocketChannel管道事件轮询

## 实例代码（采用epoll + pipe实现）

本例我们采用 epoll + pipe的机制，简单的模拟一个通信唤醒的场景来理解epoll通信，更深入的理解，可以阅读libevent源码

1. 创建管道
2. 创建epll事件：并设置关注的事件类型
3. 创建epoll
4. 往epoll注册事件
5. fork一个子进程，sleep(3)秒后，往管道写入数据
6. 父进程挂起，监听事件源(管道读取端fd)事件

```
#include <iostream>
#include <unistd.h>
#include <errno.h>
#include <sys/epoll.h>
#include <cstdio>
#include <cstdlib>
#include <cstring>

using namespace std;

int main() {
    //最大事件数
    const int MAXEVENTS = 1024;

    int ret;

    /**
     * 1. 创建管道，并将管道两端的fd存放在数组pipe_fd中
     * pipe_fd[0] ： 管道输出端，可读取fd
     * pipe_fd[1] ： 管道输入端，可写入fd
     */
    int pipe_fd[2];
    if ((ret = pipe(pipe_fd)) < 0) {
        cout << "create pipe fail:" << ret << ",errno:" << errno << endl;
        return -1;
    }
    /**
     * 2. 创建epoll关注事件源的事件类型
     */
    struct epoll_event ev;
    //设置监听事件源
    ev.data.fd = pipe_fd[0];
    //设置监听什么事件：EPOLLIN, 事件源可读事件， EPOLLET 边缘触发模式(Edge Triggered)
    ev.events = EPOLLIN | EPOLLET;

    /**
     * 3. 创建epoll对象，返回epoll对象的fd地址
     */
    int epfd = epoll_create(MAXEVENTS);
    /**
     * 4. 往epoll对象中 添加/修改/删除 一个事件(事件源fd, 事件类型&ev)
     * EPOLL_CTL_ADD：注册新的fd到epfd中；
     * EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
     * EPOLL_CTL_DEL：从epfd中删除一个fd；
     */
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, pipe_fd[0], &ev);
    //校验
    if (ret != 0) {
        cout << "epoll_ctl fail:" << ret << ",errno:" << errno << endl;
        close(pipe_fd[0]);
        close(pipe_fd[1]);
        close(epfd);
        return -1;
    }
    //fork一个进程，来读取epoll事件
    int pid = fork();

    if (pid > 0) {//父进程
        //监听事件数组
        struct epoll_event events[MAXEVENTS];
        /**
         * 6. 使用epoll，开始监听事件，并将事件放置到数组events中
         *
         * epoll_wait会阻塞当前线程，当监听的事件发生的时候，会唤醒该线程
         *
         */
        int count = epoll_wait(epfd, events, MAXEVENTS, 5000);
        char r_buf[100];
        for (int i = 0; i < count; i++) {
            //校验
            if ((events[i].data.fd == pipe_fd[0]) && (events[0].events & EPOLLIN)) {
                int r_num = read(pipe_fd[0], r_buf, 100);
                printf("parrent read num is %d bytes data from the pipe, value is %d \n", r_num, atoi(r_buf));
            }
        }
        close(pipe_fd[1]);
        close(pipe_fd[0]);
        close(epfd);
        cout << "parent close read fd[0], wirte fd[1] and epfd over" << endl;

    } else if (pid == 0) {//子进程
        //子进程不进行读取操作，关闭读取fd
        close(pipe_fd[0]);//read
        cout << "sub does't read, so close read fd[0], over" << endl;

        //当前线程睡眠3秒
        sleep(3);

        /**
         * 5. 子进程开始向管道写入数据，触发EPOLLIN事件
         */
        char w_buf[100];
        strcpy(w_buf, "1234");
        if (write(pipe_fd[1], w_buf, 5) != -1)//you can remove this line for learn
            printf("sub write write num 1234, over \n");
        //关闭写入端管道fd
        close(pipe_fd[1]);//write
        printf("sub close write fd[1] over \n");

    } else { //pid<0, fork error
        close(pipe_fd[0]);
        close(pipe_fd[1]);
        close(epfd);
        cout << "fork error:" << pid << ", close fds over" << endl;
    }

    return 0;
}
```

## 改用epoll + eventfd来实现

eventfd在2.6.22引入，是比pipe更高效的线程间事件通知机制，它的缓冲区只有8个字节。

eventfd 只能用于线程间、父子进程间的通信

```
//
// Created by milo on 17-9-18.
//
#include <iostream>
#include <unistd.h>
#include <errno.h>
#include <sys/epoll.h>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <sys/eventfd.h>

using namespace std;

int main() {

    const int MAXEVENTS = 1024;

    int ret;

    /**
     * eventfd (unsigned int __count, int __flags)
     *
     * man: http://man7.org/linux/man-pages/man2/eventfd.2.html
     *
     * initval: eventfd设备初始值
     * flags: 在2.6.26之前的版本，必须设置为0，之后的版本有以下值：
     *     EFD_SEMAPHORE
     *     EFD_CLOEXEC
     *     EFD_NONBLOCK
     */
    int efd = eventfd(0, 0);

    /**
     * 1. 创建epoll关注事件源的事件类型
     */
    struct epoll_event ev;
    //设置监听事件源
    ev.data.fd = efd;
    //设置监听什么事件：EPOLLIN, 事件源可读事件， EPOLLET 边缘触发模式(Edge Triggered)
    ev.events = EPOLLIN | EPOLLET;

    /**
     * 2. 创建epoll对象，返回epoll对象的fd地址
     */
    int epfd = epoll_create(MAXEVENTS);
    /**
     * 3. 往epoll对象中 添加/修改/删除 一个事件(事件源fd, 事件类型&ev)
     * EPOLL_CTL_ADD：注册新的fd到epfd中；
     * EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
     * EPOLL_CTL_DEL：从epfd中删除一个fd；
     */
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &ev);
    //校验
    if (ret != 0) {
        cout << "epoll_ctl fail:" << ret << ",errno:" << errno << endl;
        close(efd);
        close(epfd);
        return -1;
    }
    //fork一个进程，来读取epoll事件
    int pid = fork();

    if (pid > 0) {//父进程
        //监听事件数组
        struct epoll_event events[MAXEVENTS];
        /**
         * 4. 使用epoll，开始监听事件源，并将事件放置到数组events中
         *
         * epoll_wait会阻塞当前线程，当监听的事件发生的时候，会唤醒该线程
         *
         */
        int count = epoll_wait(epfd, events, MAXEVENTS, 5000);
        for (int i = 0; i < count; i++) {
            //校验
            if ((events[i].data.fd == efd) && (events[0].events & EPOLLIN)) {
                eventfd_t r_num;
                ssize_t size = read(efd, &r_num, sizeof(eventfd_t));
                printf("parrent read num is %d bytes data from the eventfd, value is %d \n", size, r_num);
            }
        }
        close(efd);
        close(epfd);
        cout << "parent close eventfd and epfd over" << endl;

    } else if (pid == 0) {//子进程
        printf("sub sleep 3 senconds, and then write\n");
        flush(cout);
        //当前线程睡眠3秒
        sleep(3);

        //子进程开始向eventfd设备写入数据
        eventfd_t num = 18;
        if (write(efd, &num, sizeof(eventfd_t)) == sizeof(eventfd_t))//you can remove this line for learn
            printf("sub write num %d, over \n", num);
        //关闭子进程eventfd
        close(efd);//write
        printf("sub close write eventfd over \n");

    } else { //pid<0, fork error
        close(efd);
        close(epfd);
        cout << "fork error:" << pid << ", close fds over" << endl;
    }

    return 0;
}
```

## 参考链接

[Libevent深入浅出](https://aceld.gitbooks.io/libevent/content/)

[实现机制详解](http://blog.csdn.net/u010657219/article/details/44061629)

[IO多路复用之epoll总结](http://www.cnblogs.com/Anker/p/3263780.html)

[我读过的最好的epoll讲解–转自”知乎“](http://yaocoder.blog.51cto.com/2668309/888374)

[man2:eventfd](http://man7.org/linux/man-pages/man2/eventfd.2.html)





# 参考

https://www.molingyu.com/2017/09/18/%E4%B8%80%E4%B8%AAepoll%E5%AE%9E%E4%BE%8B/

