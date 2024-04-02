# 参考

https://zhuanlan.zhihu.com/p/24614363



https://www.jianshu.com/p/646d98cbd573



https://cloud.tencent.com/developer/article/1650116



《架构整洁之道》

https://cloud.tencent.com/developer/article/1520178





# 设计原则

想必大家在学习面向对象的时候,都学习过下面几大原则:

- SRP 单一职责:该设计原则是基于康威定律的推论，每个软件模块有且只有一个被更改的理由。
- OCP 开闭原则:对扩展开放，对修改关闭。
- LSP 里氏替换原则:任何基类可以出现的地方，子类一定可以出现。
- ISP 接口隔离原则:在设计中需要避免不需要的依赖。
- DIP 依赖反转原则:高层策略性代码不应该依赖底层细节的代码，而应该是底层细节代码依赖高层策略。

这五个原则也被称为，SOLID原则取的是他们的首字母。这个也是我们做一个好设计的基础，接下来会依次对其进行解释

# SOLID原则简介

| 首字母 | 指代         | 概念                                                 |
| :----- | :----------- | :--------------------------------------------------- |
| S      | 单一功能原则 | 对象应该仅具有一种单一功能                           |
| O      | 开闭原则     | 软件体应该是对于扩展开放的，但是对于修改封闭的       |
| L      | 里氏替换原则 | 程序中对象在不改变程序正确性的前提下被它的子类所替换 |
| I      | 接口隔离原则 | 多个特定客户端接口要好于一个宽泛用途的接口           |
| D      | 依赖反转原则 | 依赖于抽象而不是一个实例                             |





# 一、单一职责原则(Single Responsibility Principle)

单一职责原则，英文全称：（Single responsibility principle），简称SRP

## 1.1 定义

> 一个类应该仅有一个引起它变化的原因，变化的方向隐含类的责任

## 1.2 优点

降低类的复杂度、提高类的可读性，提高系统的可维护性、降低变更引起的风险



在面向对象编程领域中，单一功能原则（Single responsibility principle）规定每个类都应该有一个单一的功能，并且该功能应该由这个类完全封装起来。所有它的（这个类的）服务都应该严密的和该功能平行（功能平行，意味着没有依赖）。

这个术语由罗伯特·C·马丁（Robert Cecil Martin）在他的《敏捷软件开发，原则，模式和实践》一书中的一篇名为〈面向对象设计原则〉的文章中给出。 马丁表述该原则是基于的《结构化分析和系统规格》一书中的内聚原则（Cohesion）上。

马丁把功能（职责）定义为：“改变的原因”，并且总结出一个类或者模块应该有且只有一个改变的原因。一个具体的例子就是，想象有一个用于编辑和打印报表的模块。这样的一个模块存在两个改变的原因。第一，报表的内容可以改变（编辑）。第二，报表的格式可以改变（打印）。这两方面会的改变因为完全不同的起因而发生：一个是本质的修改，一个是表面的修改。单一功能原则认为这两方面的问题事实上是两个分离的功能，因此他们应该分离在不同的类或者模块里。把有不同的改变原因的事物耦合在一起的设计是糟糕的。

保持一个类专注于单一功能点上的一个重要的原因是，它会使得类更加的健壮。继续上面的例子，如果有一个对于报表编辑流程的修改，那么将存在极大的危险性，因为假设这两个功能存在于同一个类中，修改报表的编辑流程会导致公共状态或者依赖关系的改变，打印功能的代码会因此不工作。

示例：

```
abstract class Employee {
  // This needs to be implemented
  abstract calculatePay (): number;
  // This needs to be implemented
  abstract reportHours (): number;
  // let's assume THIS is going to be the 
  // same algorithm for each employee- it can
  // be shared here.
  protected save (): Promise<any> {
    // common save algorithm
  }
}

class HR extends Employee {
  calculatePay (): number {
    // implement own algorithm
  }
  reportHours (): number {
    // implement own algorithm
  }
}

class Accounting extends Employee {
  calculatePay (): number {
    // implement own algorithm
  }
  reportHours (): number {
    // implement own algorithm
  }

}

class IT extends Employee {
  ...
}
```





## 3.SRP:单一职责

SRP很容易被大家从字面意思无界，并不是每个模块只做一个事，而是**每个模块的变化原因只有一个**。在书中对于SRP最后的解释是:

> 任何一个软件模块都应该只对某一类行为者(有共同需求的人)负责。

这里的软件模块指的就是一个源代码文件或者一组紧密相关的函数和数据结构。SRP原则应该是大家运用得最多的原则之一。在书中举了一个例子，有一个Employee类其中有三个函数:

- calculatePay():计算工资，由财务部门制定，需要向CFO汇报。
- reportHours():计算工时，人力资源制定，向COO汇报
- save():由DBA制定，向CTO汇报。

这里三个函数都放在了Employee类中，其实也就是把三个行为者的行为都耦合在了一起。一般来说计算工资，会获取正常工时，而计算工时也会获取工时，这两个函数都依赖了一个获取工时的方法，如果财务部门计算工资时，想修改逻辑，看大家辛苦了1个小时当1.1个小时发工资，这个时候修改了这个获取工时的方法，但是HR部门并不需要这个修改，这个时候就会导致reportHours()这个方法出现数据错误。所以这个时候就需要将不同行为者的代码就行拆分。

### **3.1 如何解决**

在书中给出了第一个解决方法：

![image-20210505213919142](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210505213939.png)

设计出三个类，每个类都只与一个行为者相关。这种问题的坏处是，程序员需要在程序里处理三个类，这里还介绍了使用门面模式的方法，让我们只需要在我们使用的地方使用一个类即可:

![image-20210505213934535](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210505213946.png)

这样的话我们就不需要关心其他三个类，直接调用门面模式的方法即可。

### **3.2 实际场景**

在实际场景中微服务可以算作是SRP的思想，虽然每一个微服务不止一个类，但是其整个服务也可以看做是一个模块，而每个一个模块基本也只于一个行为者相关。在我们的代码中可以使用3.1中所描述的方法来进行SRP的实现。

SRP的好处:

- 修改代码容易，由于不需要考虑修改代码是否会影响其他业务所以是很容易的。
- 更加容易维护，维护一个什么逻辑都有的代码明显比维护一个单一职责的代码难得多。
- 容易发现问题，当出现问题的时候，由于职责清晰，可以比较容易的定位。
- 松耦合，职责分离，耦合程度比较低。



# 二、开闭原则

开闭原则，英文全称：（Open Closed Principle），简称为OCP

#### 2.1 定义

> 一个软件实体如类、模块和函数应该对扩展开放，对修改关闭。
>
> 用抽象构建框架，用实现扩展细节

#### 2.2 优点

提高软件系统的可复用性及维护性



在这本书中讲述OCP可能和大家从一些资料上面看的有点不同。一般大家所认为的开闭原则，应该将那些容易变化的部分进行抽象，利用对抽象的多个实现来进行对扩展开放，而不是直接在类中去修改。

这里我用吃饭的例子来列举:

![image-20210505214044951](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210505214142.png)

每个人一天都会吃三餐，早餐，午餐，晚餐，但是随着时代的进步，又出现了下午茶，宵夜等，现在一天就不止三餐，那么其实我们就需要在这个类中去添加喝下午茶方法，吃宵夜方法，这样就导致我们没增加一个餐的种类就需要添加一个方法，在将SRP的时候我们有个例子，在同一个类中修改方法的时候容易修改其他业务逻辑，在我们这个例子中我们也会出现这个问题。怎么解决呢？那么我们就可以将变化的部分抽象出来:

![image-20210505214103640](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210505214136.png)

，后续如果还需要增加吃的方法那么只需要实现这个接口即可。

但是在这本书中，并没有去强调将变化的部分抽象出来，其认为修改是不可避免的，所以我们需要把控好修改的影响，所以提出了高层组件的修改不会影响底层组件，组件层次越低越稳定。对于J2EE的开发者来说，三层开发肯定并不陌生，controller,service,dao:

![image-20210505214116222](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210505214130.png)

如果我们修改controller那么service其实是无感知的，不会受影响，如果我们修改service，dao是不会受影响的，但是我们的controller是会受影响。所以越底层的组件那么其实应该越稳定。通过这种方式我们可以控制修改范围的影响。总结起来就是通过将系统划分为一系列组件，并且将这些组件间的依赖关系按层次结构进行组织，使得高阶组件不会因低阶组件被修改而受到影响。





# 三、Liskov替换原则

https://blog.csdn.net/tjiyu/article/details/76551307



Liskov替换原则，英文全称：（Liskov Substitution Principle），简称：LSP

#### 3.1 原则定义

> 子类必须能够替换它们的基类（IS-A）的关系

引申意义：**子类可以扩展父类的功能，但不能改变父类原有的功能**。

一个软件实体如果适用一个父类的话，那一定适用于其子类，所有引用父类的地方必须能透明地使用其子类的对象，子类对象能够替换父类对象，而程序逻辑不变。

#### 3.2 注意点

- 1.当子类的方法重载父类的方法时，方法的前置条件（即方法的输入、入参）要比父类方法的输入参数更宽松
- 2.当子类的实现父类的方法（重写、重载或者实现抽象方法），方法的后置条件（即方法的输出、返回值）要比父类更严格或相等

#### 3.3 优点

约束继承泛滥、开闭原则的一种体现

加强程序的健壮性，同时变更时也可以做到非常好的兼容提高程序的维护性、扩展性。降低需求变更时引入的风险。

# 四、接口隔离原则

接口隔离原则，英文全称：（Interface Segregation Principle），简称为：ISP

#### 4.1 定义

> 不应该强迫客户程序依赖它不用的方法

#### 4.2 引申含义

- 1.一个类对一个类的依赖应该建立在最小的接口上
- 2.建立单一接口，不要建立庞大臃肿的接口
- 3.尽量细化接口，接口中的方法尽量少
- 4.注意适度原则，一定要适度

#### 4.3 优点

符合我们常说的高内聚低耦合的设计思想从而使得类具有很好的可读性、可扩展性和可维护性





**接口隔离原则（英语：interface-segregation principles， 缩写：ISP）指明客户（client）应该不依赖于它不使用的方法**。接口隔离原则(ISP)拆分非常庞大臃肿的接口成为更小的和更具体的接口，这样客户将会只需要知道他们感兴趣的方法。这种缩小的接口也被称为角色接口（role interfaces）。接口隔离原则(ISP)的目的是系统解开耦合，从而容易重构，更改和重新部署。

## 举例

以商家接入移动支付API的场景举例，支付宝支持收费和退费；微信接口只支持收费。

```
interface PayChannel {
    void charge();
    void refund();
}

class AlipayChannel implements PayChannel {
    public void charge() {
        ...
    }
    
    public void refund() {
        ...
    }
}

class WeChatChannel implements payChannel {
    public void charge() {
        ...
    }
    
    public void refund() {
        // 没有任何代码
    }
}
```

第二种支付渠道，根本没有退款的功能，但是由于实现了PayChannel，又不得不将refund()实现成了空方法。那么，在调用中，这个方法是可以调用的，实际上什么都没有做!

将PayChannel拆成各包含一个方法的两个接口PayableChannel和RefundableChannel。



# 五、依赖倒置原则

依赖倒置原则，英文全称Dependence Inversion Principle，简写为DIP

#### 5.1 原则定义

> 高层模块(稳定)不应该依赖于低层模块（变化），二者都应该依赖于抽象（稳定）
>
> 抽象（稳定）不应该依赖于实现细节（变化），实现细节应该依赖于抽象（稳定）
>
> 针对接口编程，不要针对实现编程

#### 5.2 示例代码

例如订单模块，刚开始使用SqlServer做为数据库，代码如下：



```csharp
public class SqlServerDao {
    public void insertOrder() {
        System.out.println("通过SqlServer插入数据");
    }
}
```



```cpp
public class OrderService {

    private SqlServerDao dao = new SqlServerDao();

    public void insertOrder() {
        dao.insertOrder();
    }
}
```

没过多久，BOSS觉得SqlServer太贵了，要统一切换到MySQL上，这时候你改成下面代码：



```csharp
public class MySqlDao {
    public void insertOrder() {
        System.out.println("通过MySql插入数据");
    }
}
```



```cpp
public class OrderService {

    private MySqlDao dao = new MySqlDao();

    public void insertOrder() {
        dao.insertOrder();
    }
}
```

上面的例子OrderService就是高层模块，MySqlDao、SqlServerDao就是底层模块。高层模块不应该依赖于底层模块，而是依赖于抽象。这时候需要加入数据库的抽象层。



```csharp
public interface AbstractDao {
    /**
     * 插入订单
     */
    void insertOrder();
}
```



```cpp
public class OrderService {

    private AbstractDao dao;

    public void setDao(AbstractDao dao) {
        this.dao = dao;
    }

    public void insertOrder() {
        dao.insertOrder();
    }
}
```



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210505212856)



#### 5.3 优点

可以减少类间的耦合性、提高系统稳定性，提高代码可读性和可维护性，可降低修改程序所造成的风险





# 六、迪米特原则（最少知道、降低耦合）

