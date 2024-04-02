---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 桥接模式

## 1.1 名称
桥接模式(Bridge Design Pattern)，将**抽象部分**和它的**实现部分**分离，使它们都可以独立地变化。

这种模式涉及到一个作为**桥接的接口**，使得实体类的功能独立于接口实现类。这两种类型的类可被结构化改变而互不影响。


桥接模式，又称为桥梁模式，是结构型设计模式之一。在现实生活中大家都知道“桥梁”是连接河道两岸的主要交通枢纽，简而言之其作用就是连接河流的两边，而我们的桥接模式与现实中的情况很相似，也是承担着连接“两边”的作用。

举例：墙上的开关，可以看到的开关是**抽象**的，不用管里面具体怎么实现的。

## 1.2 问题
主要解决在有多种可能变化的情况下，用继承会造成类爆炸的问题，扩展起来不灵活。

桥接模式解决的是类属性多维度变化问题，把类的抽象层次结构和类的实现层级结构解耦，这个解耦主要指的是子类继承层面，通过组合的形式，实现各个属性类可以独立变化。

## 1.3 解决方案
### 1.3.1 结构
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202210281717117.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202210281715282.png)
##### 1.3.1.1.1 角色介绍
-   **Abstraction**: 抽象部分。  
    该类保持一个对实现部分对象的引用，抽象部分中的方法需要调用实现部分的对象来实现，该类一般为抽象类。
-   **RefinedAbstraction**: 优化的抽象部分。   （通过 **extends** 继承）
    抽象部分的具体实现，该类一般是对抽象部分的方法进行完善和扩展。
-   **Implementor**: 实现部分。  
    可以为接口或抽象类，其方法不一定要与抽象部分中的一致，一般情况下是由实现部分提供基本的操作，而抽象部分定义的则是基于实现部分这些基本操作的业务方法。
-   **ConcreteImplementorA/ConcreteImplementorB**: 实现部分的具体实现。  
    完善实现部分中方法定义的具体逻辑。
-   **Client**: 客户类，客户端程序。


### 1.3.2 例子1
我们有一个作为桥接实现的 _DrawAPI_ 接口和实现了 _DrawAPI_ 接口的实体类 _RedCircle_、_GreenCircle_。_Shape_ 是一个抽象类，将使用 _DrawAPI_ 的对象。_BridgePatternDemo_ 类使用 _Shape_ 类来画出不同颜色的圆。

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202210281734073.png)

#### 1.3.2.1 步骤1-创建桥接实现接口
```java
public interface DrawAPI { 
	public void drawCircle(int radius, int x, int y);
}
```

#### 1.3.2.2 步骤2-创建实现了 _DrawAPI_ 接口的实体桥接实现类 
RedCircle.java
```java
public class RedCircle implements DrawAPI { 
	@Override 
	public void drawCircle(int radius, int x, int y) { 
          System.out.println("Drawing Circle[ color: red, radius: " + radius +", x: " +x+", "+ y +"]"); 
	} 
}
```

GreenCircle.java
```java
public class GreenCircle implements DrawAPI {  
    @Override  
    public void drawCircle(int radius, int x, int y) {  
        System.out.println("Drawing Circle[ color: green, radius: "  
                + radius + ", x: " + x + ", " + y + "]");  
    }  
}
```

#### 1.3.2.3 步骤3-使用 _DrawAPI_ 接口创建抽象类 _Shape_
Shape.java
```java
public abstract class Shape {  
    protected DrawAPI drawAPI;  
    protected Shape(DrawAPI drawAPI){  
        this.drawAPI = drawAPI;  
    }  
    public abstract void draw();  
}
```

#### 1.3.2.4 步骤4-创建实现了 _Shape_ 抽象类的实体类
Circle.java
```java
public class Circle extends Shape {  
    private int x, y, radius;  
  
    public Circle(int x, int y, int radius, DrawAPI drawAPI) {  
        super(drawAPI);  
        this.x = x;  
        this.y = y;  
        this.radius = radius;  
    }  
  
    public void draw() {  
        drawAPI.drawCircle(radius,x,y);  
    }  
}
```


#### 1.3.2.5 步骤5-使用 _Shape_ 和 _DrawAPI_ 类画出不同颜色的圆
BridgePatternDemo.java
```java
public class BridgePatternDemo {  
    public static void main(String[] args) {  
        Shape redCircle = new Circle(100,100, 10, new RedCircle());  
        Shape greenCircle = new Circle(100,100, 10, new GreenCircle());  
  
        redCircle.draw();  
        greenCircle.draw();  
    }  
}
```

#### 1.3.2.6 步骤6-执行程序，输出结果
```
Drawing Circle[ color: red, radius: 10, x: 100, 100]
Drawing Circle[  color: green, radius: 10, x: 100, 100]
```

### 1.3.3 例子2

![Bridge](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210623155645.png)



汽车可按品牌分（本例中只考虑BMT，BenZ，Land Rover），也可按手动档、自动档、手自一体来分。如果对于每一种车都实现一个具体类，则一共要实现3 * 3=9个类。

使用继承方式的类图如下
[![Bridge](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210623155627.png)

从上图可以看到，对于每种组合都需要创建一个具体类，如果有N个维度，每个维度有M种变化，则需要$M^N$个具体类，类非常多，并且非常多的重复功能。

如果某一维度，如Transmission多一种可能，比如手自一体档（AMT），则需要增加3个类，BMWAMT，BenZAMT，LandRoverAMT。

#### 1.3.3.1 抽象车
```
package com.jasongj.brand;

import com.jasongj.transmission.Transmission;

public abstract class AbstractCar {

  protected Transmission gear;
  
  public abstract void run();
  
  public void setTransmission(Transmission gear) {
    this.gear = gear;
  }
  
}
```

按品牌分，BMW牌车

```
package com.jasongj.brand;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class BMWCar extends AbstractCar{

  private static final Logger LOG = LoggerFactory.getLogger(BMWCar.class);
  
  @Override
  public void run() {
    gear.gear();
    LOG.info("BMW is running");
  };

}
```

BenZCar

```
package com.jasongj.brand;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class BenZCar extends AbstractCar{

  private static final Logger LOG = LoggerFactory.getLogger(BenZCar.class);
  
  @Override
  public void run() {
    gear.gear();
    LOG.info("BenZCar is running");
  };

}
```

LandRoverCar

```
package com.jasongj.brand;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LandRoverCar extends AbstractCar{

  private static final Logger LOG = LoggerFactory.getLogger(LandRoverCar.class);
  
  @Override
  public void run() {
    gear.gear();
    LOG.info("LandRoverCar is running");
  };

}
```

#### 1.3.3.2 抽象变速器

```
package com.jasongj.transmission;

public abstract class Transmission{

  public abstract void gear();

}
```

手动档
```
package com.jasongj.transmission;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Manual extends Transmission {

  private static final Logger LOG = LoggerFactory.getLogger(Manual.class);

  @Override
  public void gear() {
    LOG.info("Manual transmission");
  }
}
```

自动档

```
package com.jasongj.transmission;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Auto extends Transmission {

  private static final Logger LOG = LoggerFactory.getLogger(Auto.class);

  @Override
  public void gear() {
    LOG.info("Auto transmission");
  }
}
```


有了变速器和品牌两个维度各自的实现后，可以通过聚合，实现不同品牌不同变速器的车，如下

```
package com.jasongj.client;

import com.jasongj.brand.AbstractCar;
import com.jasongj.brand.BMWCar;
import com.jasongj.brand.BenZCar;
import com.jasongj.transmission.Auto;
import com.jasongj.transmission.Manual;
import com.jasongj.transmission.Transmission;

public class BridgeClient {

  public static void main(String[] args) {
    Transmission auto = new Auto();
    AbstractCar bmw = new BMWCar();
    bmw.setTransmission(auto);
    bmw.run();
    

    Transmission manual = new Manual();
    AbstractCar benz = new BenZCar();
    benz.setTransmission(manual);
    benz.run();
  }

}
```


## 1.4 效果
### 1.4.1 使用场景
-   如果一个系统需要在构件的抽象化角色和具体化角色之间增加更多的灵活性，避免在两个层次之间建立静态的继承关系，可以通过桥接模式使他们在抽象层建立一个关联关系。
-   对于那些**不希望使用继承或因为多层次继承**导致系统类的个数急剧增加的系统，也可以考虑使用桥接模式。
-   一个**类存在两个独立变化的维度**，且这两个维度都需要进行扩展。




#### 1.4.1.1 Android 中使用
##### 1.4.1.1.1 ListView
桥接模式在Android中应用得相当广泛，但一般而言都是作用于大范围的，我们可以在源码中很多地方看到桥接模式的应用。例如我们常用的**Adapter**跟**AdapterView**之间就是一个桥接模式。
 首先ListAdapter.java：


```java
public interface ListAdapter extends Adapter{
     //继承自Adapter，扩展了自己的两个实现方法
     public boolean areAllItemsEnabled();
     boolean isEnabled(int position);
}
```

这里先来看一下父类AdapterView:

```java
public abstract class AdapterView<T extends Adapter> extends ViewGroup {
     //这里需要一个泛型的
    Adapter public abstract T getAdapter();
    public abstract void setAdapter(T adapter);
 }
```

接着来看ListView的父类AbsListView，继承自AdapterView

```java
public abstract class AbsListView extends AdapterView<ListAdapter>
     //继承自AdapterView,并且指明了T为ListAdapter
     /** * The adapter containing the data to be displayed by this view */
     ListAdapter mAdapter;
     //代码省略
     //这里实现了setAdapter的方法，实例了对实现化对象的引用
     public void setAdapter(ListAdapter adapter) {
     //这的adapter是从子类传入上来，也就是listview，拿到了具体实现化的对象
     if (adapter != null) {
         mAdapterHasStableIds = mAdapter.hasStableIds();
         if (mChoiceMode != CHOICE_MODE_NONE && mAdapterHasStableIds &&   mCheckedIdStates == null) {
             mCheckedIdStates = new LongSparseArray<Integer>();
         }
     }
     if (mCheckStates != null) {
         mCheckStates.clear();
    }
     if (mCheckedIdStates != null) {
         mCheckedIdStates.clear();
    }
 }
```

大家都知道，构建一个listview，adapter中最重要的两个方法，getCount()告知数量，getview()告知具体的view类型，接下来看看AbsListView作为一个视图的集合是如何来根据实现化对象adapter来实现的具体的view呢？

```java
protected void onAttachedToWindow() {
    super.onAttachedToWindow();
     //省略代码
     //这里在加入window的时候，getCount()确定了集合的个数
     mDataChanged = true;
     mOldItemCount = mItemCount;
     mItemCount = mAdapter.getCount();
     
 }
```

接着来看

```java
View obtainView(int position, boolean[] isScrap) {
     //代码省略
     //这里根据位置显示具体的view,return的child是从持有的实现对象mAdapter里面的具体实现的
     //方法getview来得到的。
     final View child = mAdapter.getView(position, scrapView, this);
     //代码省略 return child;
 }
```

接下来在ListView中，onMeasure调用了obtainView来确定宽高，在扩展自己的方法来排列这些view。知道了

这些以后，我们来画一个简易的UML图来看下:

![image-20210624143704439](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210624143704.png)

以上就是Android源码中的桥接模式实现

##### 1.4.1.1.2 View与Canvas
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202210281711185.png)


### 1.4.2 优缺点
#### 1.4.2.1 优点
- **分离抽象和实现部分**
   桥接模式分离了抽象和实现部分，从而极大地提高了系统的灵活性。**让抽象部分和实现部分独立开来，分别定义接口，这有助于对系统进行分层，从而产生更好的结构化的系统。**对于系统的高层部分，只需要知道抽象部分和实现部分的接口就可以了。
- **灵活的扩展性**
   由于桥接模式把抽象和实现部分分离开了，而且分别定义接口，这就使得抽象部分和实现部分可以分别独立的扩展，而不会相互影响，从而大大的提高了系统的可扩展性。可动态切换实现。
   由于桥接模式把抽象和实现部分分离开了，那么在实现桥接的时候，就可以实现动态的选择和使用具体的实现，**也就是说一个实现不再是固定的绑定在一个抽象接口上了，可以实现运行期间动态的切换实现**。

#### 1.4.2.2 缺点
- **不容易设计**
   对开发者要有一定的经验要求

## 1.5 参考
https://www.runoob.com/design-pattern/bridge-pattern.html
https://www.jianshu.com/p/5bb60f943827