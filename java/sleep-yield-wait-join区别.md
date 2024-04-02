# 概述

只有runnable到running时才会占用cpu时间片，其他都会出让cpu时间片。
线程的资源有不少，但应该包含**CPU资源和锁资源这两类。**
sleep(long mills)：**让出CPU资源，但是不会释放锁资源**。
wait()：**让出CPU资源和锁资源**。



## Thread.sleep和Thread.yield

1.Thread.sleep(long) 和Thread.yield()都是Thread类的静态方法，在调用的时候都是Thread.sleep(long)/ Thread.yield()的方式进行调用。

  **而join()是由线程对象来调用。**


## wait和notify和notifyAll

2.wait()和notify()、notifyAll()  这三个方法都是java.lang.Object的方法! 

Object 是java.lang.Object，**因为天天说Java是面向对象的**，所以Object是所有Java对象的超类，都实现Object的方法：

Object参考：[Java超类-java.lang.object](http://www.cnblogs.com/aspirant/p/8879917.html)

它们都是用于协调多个线程对共享数据的存取，所以必须在Synchronized语句块内使用这三个方法。前面说过Synchronized这个关键字用于保护共享数据，阻止其他线程对共享数据的存取。但是这样程序的流程就很不灵活了，如何才能在当前线程还没退出Synchronized数据块时让其他线程也有机会访问共享数据呢？此时就用这三个方法来灵活控制。 

(1) wait()方法使当前线程暂停执行并**释放对象锁**标志，让其他线程可以进入Synchronized数据块，当前线程被放入对象等待池中。

(2) 当调用 notify()方法后，将从对象的等待池中移走一个任意的线程并放到锁标志等待池中，只有锁标志等待池中的线程能够获取锁标志；如果锁标志等待池中没有线程，则notify()不起作用。 
(3) notifyAll()则从对象等待池中移走所有等待那个对象的线程并放到锁标志等待池中。 

 

## sleep和wait区别 

sleep与Wait的区别：**sleep是线程方法，wait是object方法**；看区别，主要是看CPU的运行机制：

它们的区别主要考虑两点：1.cpu是否继续执行、2.锁是否释放掉。

对于这两点，首先解释下cpu是否继续执行的含义：cpu为每个线程划分时间片去执行，每个时间片时间都很短，cpu不停地切换不同的线程，以看似他们好像同时执行的效果。

其次解释下锁是否释放的含义：锁如果被占用，那么这个执行代码片段是同步执行的，如果锁释放掉，就允许其它的线程继续执行此代码块了。 

明白了以上两点的含义，开始分析sleep和wait：

sleep **，释放cpu资源，不释放锁资源，**如果线程进入sleep的话，释放cpu资源，如果外层包有Synchronize，那么此锁并没有释放掉。

wait，**释放cpu资源，也释放锁资源，**一般用于锁机制中 肯定是要释放掉锁的，因为notify并不会立即调起此线程，因此cpu是不会为其分配时间片的，也就是说wait 线程进入等待池，cpu不分时间片给它，锁释放掉。

**(wait用于锁机制，sleep不是，这就是为啥sleep不释放锁，wait释放锁的原因，sleep是线程的方法，跟锁没半毛钱关系，wait，notify,notifyall 都是Object对象的方法，是一起使用的，用于锁机制)**

 

## 最后

1.sleep：Thread类的方法，必须带一个时间参数。**会让当前线程休眠进入阻塞状态并释放CPU（阿里面试题 Sleep释放CPU，wait 也会释放cpu，因为cpu资源太宝贵了，只有在线程running的时候，才会获取cpu片段）**，提供其他线程运行的机会且不考虑优先级，但如果有同步锁则sleep不会释放锁即其他线程无法获得同步锁 可通过调用interrupt()方法来唤醒休眠线程。

 

2.yield：**让出CPU调度**，Thread类的方法，类似sleep只是**不能由用户指定暂停多长时间 ，** 并且yield()方法**只能让同优先级的线程**有执行的机会。 yield()只是使当前线程重新回到可执行状态，所以执行yield()的线程有可能在进入**到可执行状态后**马上又被执行。调用yield方法只是一个建议，告诉线程调度器我的工作已经做的差不多了，可以让别的相同优先级的线程使用CPU了，没有任何机制保证采纳。

 

3.wait：Object类的方法(notify()、notifyAll()  也是Object对象)，必须放在循环体和同步代码块中，执行该方法的线程会释放锁，进入线程等待池中等待被再次唤醒(notify随机唤醒，notifyAll全部唤醒，线程结束自动唤醒)即放入锁池中竞争同步锁

 

4.join：一种特殊的wait，当前运行线程调用另一个线程的join方法，当前线程进入阻塞状态直到另一个线程运行结束等待该线程终止。 注意该方法也需要捕捉异常。

等待调用join方法的线程结束，再继续执行。如：t.join();//主要用于等待t线程运行结束，若无此句，main则会执行完毕，导致结果不可预测。

 

关于Java中线程的生命周期，首先看一下下面这张较为经典的图：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210821153225.png)



 

上图中基本上囊括了Java中多线程各重要知识点。掌握了上图中的各知识点，Java中的多线程也就基本上掌握了。主要包括：

## 线程的5个状态

1、新建状态（New）：当线程对象对创建后，即进入了新建状态，如：Thread t = new MyThread();

2、就绪状态（Runnable）：当调用线程对象的start()方法（t.start();），线程即进入就绪状态。处于就绪状态的线程，只是说明此线程已经做好了准备，随时等待CPU调度执行，**获取cpu 的使用权,**并不是说执行了t.start()此线程立即就会执行

3、运行状态（Running）：可运行状态(runnable)的线程获得了cpu 时间片（timeslice） ，执行程序代码。 当CPU开始调度处于就绪状态的线程时，此时线程才得以真正执行，即进入到运行状态。注：就 绪状态是进入到运行状态的唯一入口，也就是说，线程要想进入运行状态执行，首先必须处于就绪状态中；

4、阻塞状态（Blocked）：处于运行状态中的线程由于某种原因，暂时放弃对CPU的使用权，，也即让出了cpu timeslice， 停止执行，此时进入阻塞状态，直到其进入到就绪状态，才 有机会再次被CPU调用以进入到运行状态。才有机会再次获得cpu timeslice 转到运行(running)状态 根据阻塞产生的原因不同，阻塞状态又可以分为三种：

 

1.等待阻塞：运行状态中的线程执行wait()方法，使本线程进入到等待阻塞状态；，JVM会把该线程放入等待队列(waitting queue)中。

2.同步阻塞 –,运行(running)的线程在获取对象的同步锁时，若该同步锁被别的线程占用，获取synchronized同步锁失败 , 它会进入同步阻塞状态 ,则JVM会把该线程放入锁池(lock pool)中。

3.其他阻塞 – 通过调用线程的sleep()或join()或发出了I/O请求时，线程会进入到阻塞状态。当sleep()状态超时、join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入就绪状态。运行(running)的线程执行Thread.sleep(long ms)或t.join()方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时、join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入可运行(runnable)状态。

5、死亡状态（Dead）：线程执行完了或者因异常退出了run()方法，该线程结束生命周期。

**从图中可以看出，只有runnable到running时才会占用cpu时间片，其他都会出让cpu时间片。**
**线程的资源有不少，但应该包含CPU资源和锁资源这两类。**
**sleep(long mills)：让出CPU资源，但是不会释放锁资源。**
**wait()：让出CPU资源和锁资源。**
锁是用来线程同步的，sleep(long mills)虽然让出了CPU，但是不会让出锁，其他线程可以利用CPU时间片了，但如果其他线程要获取sleep(long mills)拥有的锁才能执行，则会因为无法获取锁而不能执行，继续等待。
但是那些没有和sleep(long mills)竞争锁的线程，一旦得到CPU时间片即可运行了。 

 

5. 死亡(DEAD)：线程run()、main() 方法执行结束，或者因异常退出了run()方法，则该线程结束生命周期。死亡的线程不可再次复生。

我写了个例子：

```
public class abc_test {

    public static void main(String[] args) {
        // TODO Auto-generated method stub 
        
        Thread thread =new Thread(new joinDemo());
        thread.start();
        
        for(int i=0;i<20;i++){
            
            System.out.println("主线程第"+i+"此执行！");
            
            if(i>=2){
                
                try{
                    //t1线程合并到主线程中，主线程停止执行过程，转而执行t1线程，直到t1执行完毕后继续；
                    thread.join();
                }catch(InterruptedException e){
                    e.printStackTrace();
                }
            }
        }
        
        
        

    }

}

class joinDemo implements Runnable{

    @Override
    public void run() {
        // TODO Auto-generated method stub
        
        for(int i=0;i<10;i++){
            
            System.out.println("线程1第"+i+"次执行");
        }
        
    }
    
      
     
}
```

结果为：

```
主线程第0此执行！
主线程第1此执行！
主线程第2此执行！
线程1第0次执行
线程1第1次执行
线程1第2次执行
线程1第3次执行
线程1第4次执行
线程1第5次执行
线程1第6次执行
线程1第7次执行
线程1第8次执行
线程1第9次执行
主线程第3此执行！
主线程第4此执行！
主线程第5此执行！
主线程第6此执行！
   .....
主线程第19此执行！
```

# 参考
https://www.jianshu.com/p/25e959037eed
https://blog.csdn.net/xiangwanpeng/article/details/54972952