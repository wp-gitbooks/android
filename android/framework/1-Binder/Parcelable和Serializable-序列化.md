# 问题
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210731225858.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210731214612.png)


1、不需要
2、序列化之前和序列化之后，equal是ture 深拷贝
3、Bundle中使用Parcel，跨进程传递
4、版本控制
5、
6、startActivity(Intent)序列化
activity启动--->AMS 两个不同进程
不同进程数据无法共享，通过序列化之后把数据带过去
7、序列化：跨进程调用
持久化：把数据存储下来


什么是序列化？
为什么要序列化？
Java序列化中字段不想进行序列化怎么办？
静态变量会被序列化嘛？





序列化

序列化的方式都有哪些

Serializable和Parcelable有什么区别，分别用在什么场景

为什么要区分场景，都用Serializable不行吗

除了上边两个还有别的序列化方式吗



一、android为什么要序列化？什么是序列化，怎么进行序列化



# 概述

## why

为什么要了解序列化？—— 进行Android开发的时候，无法将对象的引用传给Activities或者Fragments，我们需要将这些对象放到一个Intent或者Bundle里面，然后再传递。



根据以上对序列化、反序列化的理解，这个疑问可以翻译成，为什么需要把对象信息保存到存储媒介中并之后读取出来？
因为二中的解释，开发中有在JVM非运行的情况下或者在其他机器JVM上获取指定Java对象的需求。
例如：
1. 使用RMI（远程方法调用）
2. 网络中传递对象（Java Beans ）
3. 对象深度复制（包括当前对象包含的对象关系网）



## what

什么是序列化 —— 序列化，表示将一个对象转换成可存储或可传输的状态。序列化后的对象可以在网络上进行传输，也可以存储到本地。

### 什么是 java 序列化？什么情况下需要序列化？

> 序列化和反序列化是Java中最基础的知识点，也是很容易被大家遗忘的，虽然天天使用它，但并不一定都能清楚的说明白。我相信很多小伙伴们掌握的也就几句概念、关键字(Serializable)而已，如果深究问一下序列化和反序列化是如何实现、使用场景等，就可能不知所措了。在每次我作为面试官，考察Java基础时，通常都会问到序列化、反序列化的知识点，用以衡量其Java基础如何。当被问及Java序列化是什么？反序列化是什么？什么场景下会用到？如果不用它，会出现什么问题等，一般大家回答也就是几句简单的概念而已，有的工作好几年的应聘者甚至连概念都说不清楚，一脸闷逼。

### 什么是序列化和反序列化

**序列化是指将Java对象转换为字节序列的过程，而反序列化则是将字节序列转换为Java对象的过程**。

Java对象序列化是将实现了 Serializable 接口的对象转换成一个字节序列，能够通过网络传输、文件存储等方式传输 ，传输过程中却不必担心数据在不同机器、不同环境下发生改变，也不必关心字节的顺序或其他任何细节，并能够在以后将这个字节序列完全恢复为原来的对象(恢复这一过程称之为反序列化)。

对象的序列化是非常有趣的，因为利用它可以实现轻量级持久性，“持久性”意味着一个对象的生存周期不单单取决于程序是否正在运行，它可以生存于程序的调用之间。通过将一个序列化对象写入磁盘，然后在重新调用程序时恢复该对象，从而达到实现对象的持久性的效果。

本质上讲，序列化就是把实体对象状态按照一定的格式写入到有序字节流，反序列化就是从有序字节流重建对象，恢复对象状态。

简单说就是为了保存在内存中的各种对象的状态（也就是实例变量，不是方法），并且可以把保存的对象状态再读出来。虽然你可以用你自己的各种各样的方法来保存object states，但是Java给你提供一种应该比你自己好的保存对象状态的机制，那就是序列化


### 什么情况下需要序列化

1.  当你想把的**内存中的对象状态保存**到一个文件中或者数据库中时候；
2.  当你想用套接字在网络上**传送对象**的时候；
3.  当你想通过RMI传输对象的时候；

### 为什么需要使用序列化和反序列化

我们知道，不同进程/程序间进行远程通信时，可以相互发送各种类型的数据，包括文本、图片、音频、视频等，而这些数据都会以二进制序列的形式在网络上传送。

那么当两个Java进程进行通信时，能否实现进程间的对象传送呢？当然是可以的！如何做到呢？这就需要使用Java序列化与反序列化了。发送方需要把这个Java对象转换为字节序列，然后在网络上传输，接收方则需要将字节序列中恢复出Java对象。

我们清楚了为什么需要使用Java序列化和反序列化后，我们很自然地会想到Java序列化有哪些好处：

实现了数据的持久化，通过序列化可以把数据永久地保存到硬盘上（如：存储在文件里），实现永久保存对象。

利用序列化实现远程通信，即：能够在网络上传输对象。



## how

怎么通过序列化传输对象？

Android中Intent如果要传递类对象，可以通过两种方式实现。

- 方式一：Serializable，要传递的类实现Serializable接口传递对象，
- 方式二：Parcelable，要传递的类实现Parcelable接口传递对象。

Serializable（Java自带）：
 Serializable是序列化的意思，表示将一个对象转换成可存储或可传输的状态。序列化后的对象可以在网络上进行传输，也可以存储到本地。

Parcelable（android 专用）：
 除了Serializable之外，使用Parcelable也可以实现相同的效果，
 不过不同于将对象进行序列化，Parcelable方式的实现原理是将一个完整的对象进行分解，
 而分解后的每一部分都是Intent所支持的数据类型，这样也就实现传递对象的功能了。

**实现Parcelable的作用**

1）永久性保存对象，保存对象的字节序列到本地文件中；

2）通过序列化对象在网络中传递对象；

3）通过序列化在进程间传递对象。

**选择序列化方法的原则**

1）在使用内存的时候，Parcelable比Serializable性能高，所以推荐使用Parcelable。

2）Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC。

3）Parcelable不能使用在要将数据存储在磁盘上的情况，因为Parcelable不能很好的保证数据的持续性在外界有变化的情况下。尽管Serializable效率低点，但此时还是建议使用Serializable 。

**应用场景**

需要在多个部件(Activity或Service)之间通过Intent传递一些数据，简单类型（如：数字、字符串）的可以直接放入Intent。复杂类型必须实现Parcelable接口。



# how-使用例子

## 1 利用java自带的Serializable 进行序列化的例子

弄一个实体类 Person，利用Java自带的Serializable进行序列化



```java
package com.amqr.serializabletest.entity;
import java.io.Serializable;
/**
 * User: LJM
 * Date&Time: 2016-02-22 & 14:16
 * Describe: Describe Text
 */
public class Person implements Serializable{
    private static final long serialVersionUID = 7382351359868556980L;
    private String name;
    private int age;
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
}
```

使用，MainActivity和SecondActivity结合使用

MainActivity



```java
package com.amqr.serializabletest;
import android.app.Activity;
import android.content.Intent;
import android.os.Bundle;
import android.view.View;
import android.widget.TextView;
import com.amqr.serializabletest.entity.Person;
/**
 * 进行Android开发的时候，我们都知道不能将对象的引用传给Activities或者Fragments，
 * 我们需要将这些对象放到一个Intent或者Bundle里面，然后再传递。
 *
 *
 * Android中Intent如果要传递类对象，可以通过两种方式实现。
 * 方式一：Serializable，要传递的类实现Serializable接口传递对象，
 * 方式二：Parcelable，要传递的类实现Parcelable接口传递对象。
 *
 * Serializable（Java自带）：
 * Serializable是序列化的意思，表示将一个对象转换成可存储或可传输的状态。序列化后的对象可以在网络上进行传输，也可以存储到本地。
 *
 * Parcelable（android 专用）：
 * 除了Serializable之外，使用Parcelable也可以实现相同的效果，
 * 不过不同于将对象进行序列化，Parcelable方式的实现原理是将一个完整的对象进行分解，
 * 而分解后的每一部分都是Intent所支持的数据类型，这样也就实现传递对象的功能了。
 要求被传递的对象必须实现上述2种接口中的一种才能通过Intent直接传递。
 */
public class MainActivity extends Activity {
    private TextView mTvOpenNew;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        findViewById(R.id.mTvOpenNew).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent open = new Intent(MainActivity.this,SecondActivity.class);
                Person person = new Person();
                person.setName("一去二三里");
                person.setAge(18);
                // 传输方式一，intent直接调用putExtra
                // public Intent putExtra(String name, Serializable value)
                open.putExtra("put_ser_test", person);
                // 传输方式二，intent利用putExtras（注意s）传入bundle
                /**
                Bundle bundle = new Bundle();
                bundle.putSerializable("bundle_ser",person);
                open.putExtras(bundle);
                 */
                startActivity(open);
            }
        });
    }
}
```

SecondActivity



```java
package com.amqr.serializabletest;
import android.app.Activity;
import android.content.Intent;
import android.os.Bundle;
import android.widget.TextView;
import com.amqr.serializabletest.entity.Person;
/**
 * User: LJM
 * Date&Time: 2016-02-22 & 11:56
 * Describe: Describe Text
 */
public class SecondActivity extends Activity{
    private TextView mTvDate;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        mTvDate = (TextView) findViewById(R.id.mTvDate);
        Intent intent = getIntent();
        // 关键方法：getSerializableExtra ，我们的类是实现了Serializable接口的，所以写这个方法获得对象
        // public class Person implements Serializable
        Person per = (Person)intent.getSerializableExtra("put_ser_test");
        //Person per = (Person)intent.getSerializableExtra("bundle_ser");
        mTvDate.setText("名字："+per.getName()+"\\n"
                +"年龄："+per.getAge());
    }
}
```



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210728175936)



## 2 android专用的Parcelable的序列化的例子

我们写一个实体类，实现Parcelable接口，马上就被要求

1、复写describeContents方法和writeToParcel方法

2、实例化静态内部对象CREATOR，实现接口Parcelable.Creator 。

也就是，随便一个类实现了Parcelable接口就一开始就会变成这样子

Parcelable方式的实现原理是将一个完整的对象进行分解，而分解后的每一部分都是Intent所支持的数据类型，这样也就实现传递对象的功能了。



```java
public class Pen implements Parcelable{
    private String color;
    private int size;
    protected Pen(Parcel in) {
        color = in.readString();
        size = in.readInt();
    }
    public static final Creator<Pen> CREATOR = new Creator<Pen>() {
        @Override
        public Pen createFromParcel(Parcel in) {
            return new Pen(in);
        }
        @Override
        public Pen[] newArray(int size) {
            return new Pen[size];
        }
    };
    @Override
    public int describeContents() {
        return 0;
    }
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(color);
        dest.writeInt(size);
    }
}
```

系统已经帮我们做了很多事情，我们需要做的很简单，就写写我们自己需要的构造方法，写一下私有变量的get和set

大概变成这样子：



```java
package com.amqr.serializabletest.entity;
import android.os.Parcel;
import android.os.Parcelable;
/**
 * User: LJM
 * Date&Time: 2016-02-22 & 14:52
 * Describe: Describe Text
 */
public class Pen implements Parcelable{
    private String color;
    private int size;
    
    // 系统自动添加，给createFromParcel里面用
    protected Pen(Parcel in) {
        color = in.readString();
        size = in.readInt();
    }
    public static final Creator<Pen> CREATOR = new Creator<Pen>() {
        /**
         *
         * @param in
         * @return
         * createFromParcel()方法中我们要去读取刚才写出的name和age字段，
         * 并创建一个Person对象进行返回，其中color和size都是调用Parcel的readXxx()方法读取到的，
         * 注意这里读取的顺序一定要和刚才写出的顺序完全相同。
         * 读取的工作我们利用一个构造函数帮我们完成了
         */
        @Override
        public Pen createFromParcel(Parcel in) {
            return new Pen(in); // 在构造函数里面完成了 读取 的工作
        }
        //供反序列化本类数组时调用的
        @Override
        public Pen[] newArray(int size) {
            return new Pen[size];
        }
    };
    
    @Override
    public int describeContents() {
        return 0;  // 内容接口描述，默认返回0即可。
    }
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(color);  // 写出 color
        dest.writeInt(size);  // 写出 size
    }
    // ======分割线，写写get和set
    //个人自己添加
    public Pen() {
    }
    //个人自己添加
    public Pen(String color, int size) {
        this.color = color;
        this.size = size;
    }
    
    public String getColor() {
        return color;
    }
    public void setColor(String color) {
        this.color = color;
    }
    public int getSize() {
        return size;
    }
    public void setSize(int size) {
        this.size = size;
    }
}
```

其实说起来Parcelable写起来也不是很麻烦，在as里面，我们的一个实体类写好私有变量之后，让这个类继承自Parcelable，接下的步骤是：

1、复写两个方法，分别是describeContents和writeToParcel

2、实例化静态内部对象CREATOR，实现接口Parcelable.Creator 。 以上这两步系统都已经帮我们自动做好了

3、自己写写我们所需要的构造方法，变量的get和set

实现自Parcelable实体Bean已经写好了，接下来我们结合MainActivity和ThirdActivity来使用以下：

MainActivity



```java
package com.amqr.serializabletest;
import android.app.Activity;
import android.content.Intent;
import android.os.Bundle;
import android.view.View;
import com.amqr.serializabletest.entity.Pen;
import com.amqr.serializabletest.entity.Person;
/**
 * 进行Android开发的时候，我们都知道不能将对象的引用传给Activities或者Fragments，
 * 我们需要将这些对象放到一个Intent或者Bundle里面，然后再传递。
 *
 *
 * Android中Intent如果要传递类对象，可以通过两种方式实现。
 * 方式一：Serializable，要传递的类实现Serializable接口传递对象，
 * 方式二：Parcelable，要传递的类实现Parcelable接口传递对象。
 *
 * Serializable（Java自带）：
 * Serializable是序列化的意思，表示将一个对象转换成可存储或可传输的状态。序列化后的对象可以在网络上进行传输，也可以存储到本地。
 *
 * Parcelable（android 专用）：
 * 除了Serializable之外，使用Parcelable也可以实现相同的效果，
 * 不过不同于将对象进行序列化，Parcelable方式的实现原理是将一个完整的对象进行分解，
 * 而分解后的每一部分都是Intent所支持的数据类型，这样也就实现传递对象的功能了。
 要求被传递的对象必须实现上述2种接口中的一种才能通过Intent直接传递。
 */
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        findViewById(R.id.mTvOpenNew).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent open = new Intent(MainActivity.this, SecondActivity.class);
                Person person = new Person();
                person.setName("一去二三里");
                person.setAge(18);
                // 传输方式一，intent直接调用putExtra
                // public Intent putExtra(String name, Serializable value)
                open.putExtra("put_ser_test", person);
                // 传输方式二，intent利用putExtras（注意s）传入bundle
                /**
                 Bundle bundle = new Bundle();
                 bundle.putSerializable("bundle_ser",person);
                 open.putExtras(bundle);
                 */
                startActivity(open);
            }
        });
        // 采用Parcelable的方式
        findViewById(R.id.mTvOpenThird).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent mTvOpenThird = new Intent(MainActivity.this,ThirdActivity.class);
                Pen tranPen = new Pen();
                tranPen.setColor("big red");
                tranPen.setSize(98);
                // public Intent putExtra(String name, Parcelable value)
                mTvOpenThird.putExtra("parcel_test",tranPen);
                startActivity(mTvOpenThird);
            }
        });
    }
}
```

ThirdActivity



```java
package com.amqr.serializabletest;
import android.app.Activity;
import android.os.Bundle;
import android.widget.TextView;
import com.amqr.serializabletest.entity.Pen;
/**
 * User: LJM
 * Date&Time: 2016-02-22 & 14:47
 * Describe: Describe Text
 */
public class ThirdActivity extends Activity{
    private TextView mTvThirdDate;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_third);
        mTvThirdDate = (TextView) findViewById(R.id.mTvThirdDate);
//        Intent intent = getIntent();
//        Pen pen = (Pen)intent.getParcelableExtra("parcel_test");
        Pen pen = (Pen)getIntent().getParcelableExtra("parcel_test");
        mTvThirdDate = (TextView) findViewById(R.id.mTvThirdDate);
        mTvThirdDate.setText("颜色:"+pen.getColor()+"\\n"
                            +"大小:"+pen.getSize());
    }
}
```



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210728180034)



# 对比-Serializable 和Parcelable的对比

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210801194518.png)

## 选择方法
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210801194543.png)




android上应该尽量采用Parcelable，效率至上

## 编码上

Serializable代码量少，写起来方便

Parcelable代码多一些



## 效率上

Parcelable的速度比高十倍以上

serializable的迷人之处在于你只需要对某个类以及它的属性实现Serializable 接口即可。Serializable 接口是一种标识接口（marker interface），这意味着无需实现方法，Java便会对这个对象进行高效的序列化操作。

这种方法的缺点是使用了反射，序列化的过程较慢。这种机制会在序列化的时候创建许多的临时对象，容易触发垃圾回收。

Parcelable方式的实现原理是将一个完整的对象进行分解，而分解后的每一部分都是Intent所支持的数据类型，这样也就实现传递对象的功能了







## 1、作用

`Serializable`的作用是为了保存对象的属性到本地文件、数据库、网络流、rmi以方
便数据传输，当然这种传输可以是程序内的也可以是两个程序间的。而Android的
`Parcelable`的设计初衷是因为`Serializable`效率过慢，为了在程序内不同组件间以及
不同Android程序间(AIDL)高效的传输数据而设计，这些数据仅在内存中存在，
`Parcelable`是通过`IBinder`通信的消息的载体。

从上面的设计上我们就可以看出优劣了

## 2、效率及选择

`Parcelable`的性能比`Serializable`好，在内存开销方面较小，所以在内存间数据传输
时推荐使用`Parcelable`，如`activity`间传输数据，而`Serializable`可将数据持久化方便
保存，所以在需要保存或网络传输数据时选择`Serializable`，因为android不同版本
`Parcelable`可能不同，所以不推荐使用`Parcelable`进行数据持久化。

## 3、编程实现

对于`Serializable`，类只需要实现`Serializable`接口，并提供一个序列化版本
`id(serialVersionUID)`即可。而`Parcelable`则需要实现`writeToParcel`、
`describeContents`函数以及静态的CREATOR变量，实际上就是将如何打包和解包
的工作自己来定义，而序列化的这些操作完全由底层实现。

`Parcelable`的一个实现例子如下

```javascript
  public class MyParcelable implements Parcelable {
     private int mData;
     private String mStr;

     public int describeContents() {
        return 0;
     }

     // 写数据进行保存
     public void writeToParcel(Parcel out, int flags) {
        out.writeInt(mData);
        out.writeString(mStr);
     }

     // 用来创建自定义的Parcelable的对象
     public static final Parcelable.Creator CREATOR= new Parcelable.Creator() {
        public MyParcelable createFromParcel(Parcel in) {
           return new MyParcelable(in);
        }

        public MyParcelable[] newArray(int size) {
           return new MyParcelable[size];
        }
     };

     // 读数据进行恢复
     private MyParcelable(Parcel in) {
        mData = in.readInt();
        mStr = in.readString();
     }
  }
```

从上面我们可以看出Parcel的写入和读出顺序是一致的。如果元素是list读出时需要
先new一个`ArrayList`传入，否则会报空指针异常。如下：

```javascript
  list = new ArrayList();
  in.readStringList(list);
```

**PS:** 在自己使用时，read数据时误将前面int数据当作long读出，结果后面的顺序错乱，报如下异常，当类字段较多时务必保持写入和读取的类型及顺序一致。

```javascript
  12-21 20:14:10.317: E/AndroidRuntime(21114): Caused by: java.lan
g.RuntimeException: Parcel android.os.Parcel@4126ed60: Unmarshal
ling unknown type code 3014773 at offset 164
```

## 4、高级功能上

`Serializable`序列化不保存静态变量，可以使用`Transient`关键字对部分字段不进行序
列化，也可以覆盖`writeObject`、`readObject`方法以实现序列化过程自定义。





# 应用

## 面试场景

> Android 开发中对两个 Activity 之前传递数据，应该很熟悉吧？

嗯，当然没问题。一般采用 `Intent.putXXX()` 就可以实现各种轻量级数据的传递。

> 那对于自定义的 Object 呢？

直接使用 `Bundle` 的 `putSerializable()` 即可。需要把对象实现 `Serializable` 接口，最后使用 `Intent.putExtras(Bundle)` 把数据放进 `Intent` 即可。

> 除了这种方式，还有其它方式吗？和这种方式有什么区别呢？

我知道还有 `Bundle.putParcelable()` ，不过我们平时基本都只用 `Serializable` 方式。

> 为什么不用 `Parcelable` 方式呢？它们有什么不同呢？

因为简单呀，`Serializable` 方式只需要实现接口一句代码就好了，`Parcelable` 我记得有很多代码。对于它们的区别嘛，em......额......嗯.......





## 在两个 Activity 之间传递对象还需要注意什么呢？

**对象的大小，对象的大小，对象的大小！！！**

重要的事情说三遍，一定要注意对象的大小。`Intent` 中的 `Bundle` 是使用 `Binder` 机制进行数据传送的。能使用的 Binder 的缓冲区是有大小限制的（有些手机是 2 M），而一个进程默认有 16 个 `Binder` 线程，所以一个线程能占用的缓冲区就更小了（ 有人以前做过测试，大约一个线程可以占用 128 KB）。所以当你看到 

`The Binder transaction failed because it was too large` 这类 `TransactionTooLargeException` 异常时，你应该知道怎么解决了。




































# Parcelable和Serializable有什么用，它们有什么差别?

Parcelable和Serializable都可以实现序列化，使对象可以变为二进制流在内存中传输数据。在Android中，只要实现二者之一就可以使用Intent和Binder来传递数据。实现了Parcelable接囗的类依赖于Parcel这个类来实现数据传递，它并不是一个一般化用途的序列化机制，主要用于IPC机制下的高性能传输。

## 差别

### 1 来源上

Parcelable是Android提供的序列化接口，Serializable是Java提供的序列化接口。因此Parcelable只能在Android中使用，而Serializable可以在任何使用Java语言的地方使用。

### 2 使用上

Parcelable使用起来比较麻烦，序列化过程需要实现Parcelable的`writeToParcel(Parcel dest, int f1ags)`方法和`describeContents()`方法。其中`describeContents()`方法直接返回0就可以了。为了反序列化，还需要提供一个非空的名为`CREATOR`的静态字段，该字段类型是实现了Parcelable.Creator接口的类，一般用一个匿名内部类实现就可以了。现在也有一些插件可以方便地实现Parcelable接口。

Serializable的使用就比较简单，直接实现Serializable接口即可，该接口没有任何方法。序列化机制依赖于一个long型的`serialVersionUID`，如果没有显式的指定，在序列化运行时会基于该类的结构自动计算出一个值。如果类的结构发生变化就会导致自动计算出的`serialVersionUID`不同。这就会导致一个问题，序列化之后类如果新增了一个字段，反序列过程就会失败。一般会报**InvalidClassException**这样的异常。
而如果显式的指定了`serialversionUID`，只要类的结构不发生重大变化，如字段类型发生变化等，仅仅添加或者删除字段等都可以反序列化成功。
注意：如果要把对象持久化到存储设备或者通过网络传输到其它设备，最好使用Serializable。

### 3 效率上

Serializable的序列化和反序列化都需要使用到IO操作；而Parcelable不需要IO操作，Parcelable的效率要高于Serializable，Android中推荐使用Parcelable。

## 自定义一个类让其实现Parcelable，大致流程是什么?

自定义一个类让其实现Parcelable接口，大致流程是先实现该接口的`writeToParcel(Parcel dest, int flags)`和`describeContents()`方法。然后添加一个Parcelable.Creator类型的名字为`CREATOR`的非空字段。例如：

```
public class Person implements Parcelable {
    private String name;
    private int age;
    
    //...
    
    @Override
    public int describeContents() {
        return 0;
    }
    
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name); // 写出name
        dest.writeInt(age); // 写出age
    }
    
    public static final Parcelable.Creator<Person> CREATOR = new Parcelable.
            Creator<Person>() {
            
        @Override
        public Person createFromParcel(Parcel source) {
            Person person = new Person();
            person.name = source.readString(); // 读取name
            person.age = source.readInt(); // 读取age
            return person;
        }
        
        @Override
        public Person[] newArray(int size) {
            return new Person[size];
        }
    };
}
```

该类中的字段类型除了基本类型和String及它们对应的数组，如果有其它自定义的类型，也需要实现Parcelable或者Serializable接口。

# 面试题
## Java序列化与反序列化是什么？

Java序列化是指把Java对象转换为字节序列的过程，而Java反序列化是指把字节序列恢复为Java对象的过程：

-   **序列化：**序列化是把对象转换成有序字节流，以便在网络上传输或者保存在本地文件中。核心作用是对象状态的保存与重建。我们都知道，Java对象是保存在JVM的堆内存中的，也就是说，如果JVM堆不存在了，那么对象也就跟着消失了。
    
    而序列化提供了一种方案，可以让你在即使JVM停机的情况下也能把对象保存下来的方案。就像我们平时用的U盘一样。把Java对象序列化成可存储或传输的形式（如二进制流），比如保存在文件中。这样，当再次需要这个对象的时候，从文件中读取出二进制流，再从二进制流中反序列化出对象。
    
-   **反序列化：**客户端从文件中或网络上获得序列化后的对象字节流，根据字节流中所保存的对象状态及描述信息，通过反序列化重建对象。
    

## 为什么需要序列化与反序列化？

简要描述：**对内存中的对象进行持久化或网络传输, 这个时候都需要序列化和反序列化**

深入描述：

1.  **对象序列化可以实现分布式对象。**

主要应用例如：RMI(即远程调用Remote Method Invocation)要利用对象序列化运行远程主机上的服务，就像在本地机上运行对象时一样。

2.  **java对象序列化不仅保留一个对象的数据，而且递归保存对象引用的每个对象的数据。**

可以将整个对象层次写入字节流中，可以保存在文件中或在网络连接上传递。利用对象序列化可以进行对象的"深复制"，即复制对象本身及引用的对象本身。序列化一个对象可能得到整个对象序列。

3.  **序列化可以将内存中的类写入文件或数据库中。**

比如：将某个类序列化后存为文件，下次读取时只需将文件中的数据反序列化就可以将原先的类还原到内存中。也可以将类序列化为流数据进行传输。

总的来说就是将一个已经实例化的类转成文件存储，下次需要实例化的时候只要反序列化即可将类实例化到内存中并保留序列化时类中的所有变量和状态。

4.  **对象、文件、数据，有许多不同的格式，很难统一传输和保存。**

序列化以后就都是字节流了，无论原来是什么东西，都能变成一样的东西，就可以进行通用的格式传输或保存，传输结束以后，要再次使用，就进行反序列化还原，这样对象还是对象，文件还是文件。

## 序列化实现的方式有哪些？  [[Serializable中transient]]

实现**Serializable**接口或者**Externalizable**接口。

### Serializable接口

**类通过实现 `java.io.Serializable` 接口以启用其序列化功能**。可序列化类的所有子类型本身都是可序列化的。**序列化接口没有方法或字段，仅用于标识可序列化的语义。**

如以下例子：

import java.io.Serializable;

public class User implements Serializable {
   private String name;
   private int age;
   public String getName() {
       return name;
   }
   public void setName(String name) {
       this.name = name;
   }

   @Override
   public String toString() {
       return "User{" +
               "name='" + name +
               '}';
   }
}

通过下面的代码进行序列化及反序列化：

public class SerializableDemo {

   public static void main(String[] args) {
       //Initializes The Object
       User user = new User();
       user.setName("cosen");
       System.out.println(user);

       //Write Obj to File
       try (FileOutputStream fos = new FileOutputStream("tempFile"); ObjectOutputStream oos = new ObjectOutputStream(
           fos)) {
           oos.writeObject(user);
       } catch (IOException e) {
           e.printStackTrace();
       }

       //Read Obj from File
       File file = new File("tempFile");
       try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file))) {
           User newUser = (User)ois.readObject();
           System.out.println(newUser);
       } catch (IOException | ClassNotFoundException e) {
           e.printStackTrace();
       }
   }
}

//OutPut:
//User{name='cosen'}
//User{name='cosen'}

### Externalizable接口

`Externalizable`继承自`Serializable`，该接口中定义了两个抽象方法：`writeExternal()`与`readExternal()`。

当使用`Externalizable`接口来进行序列化与反序列化的时候需要开发人员重写`writeExternal()`与`readExternal()`方法。否则所有变量的值都会变成默认值。

public class User implements Externalizable {

   private String name;
   private int age;

   public String getName() {
       return name;
   }
   public void setName(String name) {
       this.name = name;
   }
   public void writeExternal(ObjectOutput out) throws IOException {
       out.writeObject(name);
   }
   public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
       name = (String) in.readObject();
   }

   @Override
   public String toString() {
       return "User{" +
               "name='" + name +
               '}';
   }
}

通过下面的代码进行序列化及反序列化：

public class ExternalizableDemo1 {

  public static void main(String[] args) {
      //Write Obj to file
      User user = new User();
      user.setName("cosen");
      try(ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("tempFile"))){
          oos.writeObject(user);
      } catch (IOException e) {
          e.printStackTrace();
      }

      //Read Obj from file
      File file = new File("tempFile");
      try(ObjectInputStream ois =  new ObjectInputStream(new FileInputStream(file))){
          User newInstance = (User) ois.readObject();
          //output
          System.out.println(newInstance);
      } catch (IOException | ClassNotFoundException e ) {
          e.printStackTrace();
      }
  }
}

//OutPut:
//User{name='cosen'}

### 两种序列化的对比

实现Serializable接口

实现Externalizable接口

系统自动存储必要的信息

程序员决定存储哪些信息

Java内建支持，易于实现，只需要实现该接口即可，无需任何代码支持

必须实现接口内的两个方法

性能略差

性能略好

## 什么是serialVersionUID？

serialVersionUID 用来表明类的不同版本间的兼容性

Java的序列化机制是通过在运行时判断类的serialVersionUID来验证版本一致性的。在进行反序列化时，JVM会把传来的字节流中的serialVersionUID与本地相应实体（类）的serialVersionUID进行比较，如果相同就认为是一致的，可以进行反序列化，否则就会出现序列化版本不一致的异常。

## 为什么还要显示指定serialVersionUID的值?

如果不显示指定serialVersionUID, JVM在序列化时会根据属性自动生成一个serialVersionUID, 然后与属性一起序列化, 再进行持久化或网络传输. 在反序列化时, JVM会再根据属性自动生成一个新版serialVersionUID, 然后将这个新版serialVersionUID与序列化时生成的旧版serialVersionUID进行比较, 如果相同则反序列化成功, 否则报错.

如果显示指定了, JVM在序列化和反序列化时仍然都会生成一个serialVersionUID, 但值为我们显示指定的值, 这样在反序列化时新旧版本的serialVersionUID就一致了.

在实际开发中, 不显示指定serialVersionUID的情况会导致什么问题? 如果我们的类写完后不再修改, 那当然不会有问题, 但这在实际开发中是不可能的, 我们的类会不断迭代, 一旦类被修改了, 那旧对象反序列化就会报错. 所以在实际开发中, 我们都会显示指定一个serialVersionUID, 值是多少无所谓, 只要不变就行。

## serialVersionUID什么时候修改？

《阿里巴巴Java开发手册》中有以下规定：

[![](https://camo.githubusercontent.com/c7817ac2c52603c71a14e4f269dce576748a9f4751f5dce1cb6fec3c3a4c3c30/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f696d6167652d32303231303232363232323333393630362e706e67)](https://camo.githubusercontent.com/c7817ac2c52603c71a14e4f269dce576748a9f4751f5dce1cb6fec3c3a4c3c30/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f696d6167652d32303231303232363232323333393630362e706e67)

想要深入了解的小伙伴，可以看这篇文章：[https://juejin.cn/post/6844903746682486791](https://juejin.cn/post/6844903746682486791)

## Java 序列化中如果有些字段不想进行序列化，怎么办？

对于不想进行序列化的变量，使用 transient 关键字修饰。

`transient` 关键字的作用是控制变量的序列化，在变量声明前加上该关键字，可以阻止该变量被序列化到文件中，在被反序列化后，`transient` 变量的值被设为初始值，如 int 型的是 0，对象型的是 null。transient 只能修饰变量，不能修饰类和方法。

## 静态变量会被序列化吗?

**不会。因为序列化是针对对象而言的, 而静态变量优先于对象存在, 随着类的加载而加载, 所以不会被序列化**.

看到这个结论, 是不是有人会问, serialVersionUID也被static修饰, 为什么serialVersionUID会被序列化? 其实serialVersionUID属性并没有被序列化, JVM在序列化对象时会自动生成一个serialVersionUID, 然后将我们显示指定的serialVersionUID属性值赋给自动生成的serialVersionUID。


## 为什么有些java类要实现Serializable接口

为了网络进行传输或者持久化

**什么是序列化**

将对象的状态信息转换为可以存储或传输的形式的过程

**除了实现Serializable接口还有什么序列化方式**

-   Json序列化
-   FastJson序列化
-   ProtoBuff序列化

## 什么是java序列化，如何实现java序列化？或者请解释Serializable接口的作用。

我们有时候将一个java对象变成字节流的形式传出去或者从一个字节流中恢复成一个java对象，例如，要将java对象存储到硬盘或者传送给网络上的其他计算机，这个过程我们可以自己写代码去把一个java对象变成某个格式的字节流再传输。

但是，jre本身就提供了这种支持，我们可以调用`OutputStream`的`writeObject`方法来做，如果要让java帮我们做，要被传输的对象必须实现`serializable`接口，这样，javac编译时就会进行特殊处理，编译的类才可以被`writeObject`方法操作，这就是所谓的序列化。需要被序列化的类必须实现`Serializable`接口，该接口是一个mini接口，其中没有需要实现方法，implements Serializable只是为了标注该对象是可被序列化的。

例如，在web开发中，如果对象被保存在了Session中，tomcat在重启时要把Session对象序列化到硬盘，这个对象就必须实现Serializable接口。如果对象要经过分布式系统进行网络传输，被传输的对象就必须实现Serializable接口。


## 什么是序列化，怎么序列化，为什么序列化，反序列化会遇到什么问题，如何解决

## 什么是 java 序列化？什么情况下需要序列化？

简单说就是为了保存在内存中的各种对象的状态（也就是实例变量，不是方法），并且可以把保存的对象状态再读出来。虽然你可以用你自己的各种各样的方法来保存object states，但是Java给你提供一种应该比你自己好的保存对象状态的机制，那就是序列化。

## 什么情况下需要序列化：

-   当你想把的内存中的对象状态保存到一个文件中或者数据库中时候；
-   当你想用套接字在网络上传送对象的时候；
-   当你想通过RMI传输对象的时候；
