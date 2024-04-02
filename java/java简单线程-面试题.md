# notify和NotifyAll的区别 



## 基础知识

首先我们需要知道，这几个都是Object对象的方法。换言之，Java中所有的对象都有这些方法。

```
public final native void notify();
public final native void notifyAll();
public final native void wait(long timeout) throws InterruptedException;
public final void wait() throws InterruptedException {
    wait(0);
}
```

其中notify()、notifyAll()、wait(long timeout)是**本地方法**  
其次我们需要知道这几个方法主要是用来个**线程**之间通信的。那可能就有人会问，既然是用来线程之间通信的，那为什么这几个方法不是在线程类Thread上呢？对于这个问题，我们先来看下这几个方法的具体使用方式再来回答这个问题。

## 用法

Java中规定，在调用者三个方法时，当前线程必须获得对象锁。因此就得配合synchronized关键字来使用

```
//使用模式，不代表可运行代码
synchronized(object) {
    while(contidion) {
        object.wait();
    }
    //object.notify();
    //object.notifyAll();
}
```

或者

```
public synchronized void methodName() {
    while(contidion) {
        object.wait();
    }
    //object.notify();
    //object.notifyAll();
}
```

在synchronized拿到对象锁之后，synchronized代码块或者方法中，必定是会持有对象锁的，因此就可以使用wait()或者notify()。  
通过上述使用方法，我们也能很好理解为什么这几个方法是在Object上而不是在Thread上。因为每个对象都可以作为synchronized锁的对象，因此wait、notify等必须和对象关联才能配合synchronized使用。

## 作用


|	方法	|	作用	|
|	-----	|	----	|
|	wait	|	线程自动释放占有的对象锁，并等待notify。|
|	notify	|	随机唤醒一个正在wait当前对象的线程，并让被唤醒的线程拿到对象锁	|
|	notifyAll	|	唤醒所有正在wait当前对象的线程，但是被唤醒的线程会再次去竞争对象锁。因为一次只有一个线程能拿到锁，所有其他没有拿到锁的线程会被阻塞。推荐使用。|


### 实际案例

接下来我们就使用wait()、notify()来实现一个生产者、消费者模式。这个也是面试过程中可能会被问到的地方。至于什么是生产者消费者模式，不明白的同学请自行百度。  
首先是一些基础的代码

```
private static Boolean run = true;//控制是否生产和消费
private static final Integer MAX_CAPACITY = 5;//缓冲区最大数量
private static final LinkedBlockingQueue<String> queue = new LinkedBlockingQueue<>();//缓冲队列
```

生产者代码
```
/**
 * 生产者
 */
class Producter extends Thread {
    @Override
    public void run() {
        while (run) {
            synchronized (queue) {
                while (queue.size() >= MAX_CAPACITY * 2) {
                    try {
                        System.out.println("缓冲队列已满，等待消费");
                        queue.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                try {
                    String string = UUID.randomUUID().toString();
                    queue.put(string);
                    System.out.println("生产:" + string);
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                queue.notifyAll();//通知生产者和消费者
            }
        }
    }
}
```


消费者代码
```
/**
 * 消费者
 */
class Consumer extends Thread {
    @Override
    public void run() {
        while (run) {
            synchronized (queue) {
                while (queue.isEmpty()) {
                    try {
                        System.out.println("队列为空，等待生产");
                        queue.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                try {
                    System.out.println("消费：" + queue.take());
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                queue.notifyAll();//通知生产者和消费者
            }
        }
    }
}
```



代码说明

> 1、生产者和消费者都继承了线程Thread，因为wait、notify本身就是线程间通信使用  
> 2、生产者和消费者都有两层while，外层的while是用来判断是否运行生产者和消费者。内存的while用来判断队列queue是否已满或者为空，如果满足条件，则使得当前线程变成等待状态（等待notify）。  
> 3、内层的条件判断为什么用while不用if，原因是当线程被wait之后，会释放对象锁。当等待的线程被notify之后，必须再次尝试去获取对象锁，如果没有获取到对象锁，那还必须等待，直到拿到对象锁之后才能向后执行。  
> 4、当生产者生产了一个数据或者消费者消费了一个数据之后，使用notifyAll()方法来通知所有等待当前对象锁的线程，但是一次只会有一个等待的线程能拿到锁。  
> 5、我们使用queue作为锁的对象在不同线程之间进行通信

代码运行结果

```
队列为空，等待生产
生产:e422484e-8eb3-4a91-8a7c-97e21d7ef498
生产:7894b802-2529-4798-ba98-9f0657f39240
生产:848f6759-a427-4a94-89dc-3f484daaa467
生产:f711d3dc-972c-4c44-8640-faffe376d354
生产:38a08e62-d774-4ed5-8b51-f1ad534c246f
消费：e422484e-8eb3-4a91-8a7c-97e21d7ef498
消费：7894b802-2529-4798-ba98-9f0657f39240
消费：848f6759-a427-4a94-89dc-3f484daaa467
消费：f711d3dc-972c-4c44-8640-faffe376d354
消费：38a08e62-d774-4ed5-8b51-f1ad534c246f
队列为空，等待生产
生产:9fae26f3-0b6e-4fbd-9620-040667efe0af
生产:95bb1d88-e08a-4f70-a270-f75760994184
消费：9fae26f3-0b6e-4fbd-9620-040667efe0af
消费：95bb1d88-e08a-4f70-a270-f75760994184
队列为空，等待生产
生产:13d304bc-dff3-44a4-9527-2e0facd884e7
生产:2693e069-bae1-4beb-adcd-a3c3bf5d232b
消费：13d304bc-dff3-44a4-9527-2e0facd884e7
消费：2693e069-bae1-4beb-adcd-a3c3bf5d232b
```

由结果也可以看出来，生产和消费是无规则的，因为虽然notifyAll()通知了所有的等待线程，但是不确定那个线程中能拿到对象锁。但是也有一个很明显的问题，因为同时只有一个线程能拿到对象锁，生产者和消费者不可能同时运行。


# wait()和sleep()的区别
