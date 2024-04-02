# 装饰模式

## 名称
#设计模式/扩展性 
装饰模式(Decorator Pattern) ：**动态地给一个对象增加一些额外的职责**(Responsibility)，就增加对象功能来说，**装饰模式比生成子类**实现更为灵活。
![装饰模式示例](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210623152823.png)

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210623154619.png)

## 问题
装饰器主要解决的是**直接继承**下因功能的不断横向扩展导致⼦类膨胀的问题，⽽是⽤装饰器模式后就会 ⽐直接继承显得更加灵活同时这样也就不再需要考虑⼦类的维护。

装饰模式(Decorator Pattern) ：**动态地给一个对象增加一些额外的职责**(Responsibility)，就增加对象功能来说，**装饰模式比生成子类**实现更为灵活。


## 解决方案
### 结构
* Component（**抽象组件**）：定义抽象接⼝，接口或者抽象类，被装饰的最原始的对象。具体组件与抽象装饰角色的父类。
* ConcreteComponent（具体组件）：实现抽象组件的接口，可以是一组。
* Decorator（**抽象装饰角色**）：一般是抽象类，抽象组件的子类，同时持有一个被装饰者的引用，用来调用被装饰者的方法;同时可以给被装饰者增加新的职责。
* ConcreteDecorator（具体装饰类）：抽象装饰角色的具体实现，扩展装饰具体的实现逻辑。



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210623152911)



![image-20210623153332908](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210623153333.png)


![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210401094211.png)

### 创建抽象组件

```
  public abstract class Room {
        public abstract void fitment();//装修方法
    }
```

### 创建具体组件

```
    public class NewRoom extends Room {//继承Room
        @Override
        public void fitment() {
            System.out.println("这是一间新房：装上电");
        }
    }
```

### 创建抽象装饰角色
```
     public abstract class RoomDecorator extends Room {//继承Room，拥有父类相同的方法
        private Room mRoom;//持有被装饰者的引用，这里是需要装修的房间

        public RoomDecorator(Room room) {
            this.mRoom = room;
        }

        @Override
        public void fitment() {
            mRoom.fitment();//调用被装饰者的方法
        }
    }
```

### 创建具体装饰类

```
    public class Bedroom extends RoomDecorator {//卧室类，继承自RoomDecorator

        public Bedroom(Room room) {
            super(room);
        }

        @Override
        public void fitment() {
            super.fitment();
            addBedding();
        }

        private void addBedding() {
            System.out.println("装修成卧室：添加卧具");
        }
    }

    public class Kitchen extends RoomDecorator {//厨房类，继承自RoomDecorator

        public Kitchen(Room room) {
            super(room);
        }

        @Override
        public void fitment() {
            super.fitment();
            addKitchenware();
        }

        private void addKitchenware() {
            System.out.println("装修成厨房：添加厨具");
        }
    }
```


### 测试
```java
     public void test() {
        Room newRoom = new NewRoom();//有一间新房间
        RoomDecorator bedroom = new Bedroom(newRoom);
        bedroom.fitment();//装修成卧室
        RoomDecorator kitchen = new Kitchen(newRoom);
        kitchen.fitment();//装修成厨房
    }
```

结果
```
这是一间新房：装上电
装修成卧室：添加卧具
这是一间新房：装上电
装修成厨房：添加厨具
```


## 效果
### 使用场景
- 需要**扩展一个类的功能，或给一个类增加附加功能**时
- 需要动态的给一个**对象增加功能**，这些功能可以再动态的撤销
- 当不能采用继承的方式对系统进行扩充或者采用继承不利于系统扩展和维护时。

### Android使用场景

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210623154126)

### 优缺点

#### 优点
- 采用组合的方式，可以动态的扩展功能，同时也可以在运行时选择不同的装饰器，来实现不同的功能。
- 有效避免了使用继承的方式扩展对象功能而带来的灵活性差，子类无限制扩张的问题。
- 被装饰者与装饰者解偶，被装饰者可以不知道装饰者的存在，同时新增功能时原有代码也无需改变，符合开放封闭原则。

#### 缺点
- 装饰层过多的话，维护起来比较困难。
- 如果要修改抽象组件这个基类的话，后面的一些子类可能也需跟着修改，较容易出错。





## 参考
https://www.cfanz.cn/mobile/resource/detail/rMgPQmNMLqJYP

装饰模式与继承？

装饰模式结构同桥接模式？

装饰模式扩展同代理模式？

https://blog.csdn.net/ezconn/article/details/107302087
https://blog.csdn.net/zhshulin/article/details/38665187
































