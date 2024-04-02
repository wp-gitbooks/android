# 线索
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210810224058.png)


# 概述

什么是泛型？为什么需要泛型？如何使用？是什么原理？如何改进？ 这基本上就是我们学习一项技术的正确套路，本文将按照以上顺序展开，由于水平有限，肯定会有不足之处，请多包含和指教。

## 什么是泛型

泛型的本质是**参数化类型**，即给**类型指定一个参数，然后在使用时再指定此参数具体的值**，那样这个类型就可以在使用时决定了。这种参数类型可以用在类、接口和方法中，分别被称为泛型类、泛型接口、泛型方法。


## 为什么需要泛型
Java中引入泛型最主要的目的是将**类型检查**工作提前到编译时期，将**类型强转（cast）** 工作交给编译器，从而让你在编译时期就获得类型转换异常以及去掉源码中的类型强转代码。例如


**没有泛型前：**

```java
private static void genericTest() {
    List arrayList = new ArrayList();
    arrayList.add("总有刁民想害朕");
    arrayList.add(7);

    for (int i = 0; i < arrayList.size(); i++) {
        Object item = arrayList.get(i);
        if (item instanceof String) {
            String str = (String) item;
            System.out.println("泛型测试 item = " + str);
        }else if (item instanceof Integer)
        {
            Integer inte = (Integer) item;
            System.out.println("泛型测试 item = " + inte);
        }
    }
}
```


**有了泛型后：**

```java
private static void genericTest2() {
     List<String> arrayList = new ArrayList<>();
     arrayList.add("总有刁民想害朕");
     arrayList.add(7); //..(参数不匹配：int 无法转换为String)
     ...
 }
```


如上代码，编译器在编译时期即可完成**类型检查**工作，并提出错误（其实IDE在代码编辑过程中已经报红了）。这次该编译器发飙了：你丫是不是傻，这是一个只能放入`String`类型的列表，你放`Int`类型是几个意思？



# 泛型作用的对象

泛型有三种使用方式，分别为：泛型类、泛型接口和泛型方法。



## 泛型类
在类的申明时指定参数，即构成了泛型类，例如下面代码中就指定`T`为类型参数，那么在这个类里面就可以使用这个类型了。例如申明`T`类型的变量`name`，申明`T`类型的形参`param`等操作。

```java
public class Generic<T> {
    public T name;
    public Generic(T param){
        name=param;
    }
    public T m(){
        return name;
    }
}
```

那么在使用类时就可以传入相应的类型，构建不同类型的实例，如下面代码分别传入了`String`,`Integer`,`Boolean`3个类型：

```text
private static void genericClass()
 {
     Generic<String> str=new Generic<>("总有刁民想害朕");
     Generic<Integer> integer=new Generic<>(110);
     Generic<Boolean> b=new Generic<>(true);

     System.out.println("传入类型："+str.name+"  "+integer.name+"  "+b.name);
}
```

输出结果为：`传入类型：总有刁民想害朕 110 true`

如果没有泛型，我们想要达到上面的效果需要定义三个类，或者一个包含三个构造函数，三个取值方法的类。



## 泛型接口

泛型接口与泛型类的定义基本一致

```java
public interface Generator<T> {
    public T produce();
}
```



## 泛型方法

这个相对来说就比较复杂，当我首次接触时也是一脸懵逼，抓住特点后也就没有那么难了。

```java
public class Generic<T> {
    public T name;
    public  Generic(){}
    public Generic(T param){
        name=param;
    }
    public T m(){
        return name;
    }
    public <E> void m1(E e){ }
    public <T> T m2(T e){ }
}
```

重点看`public <E> void m1(E e){ }`这就是一个泛型方法，判断一个方法是否是泛型方法关键看方法返回值前面有**没有使用`<>`标记的类型**，有就是，没有就不是。这个`<>`里面的类型参数就相当于为这个方法声明了一个类型，这个类型可以在此方法的作用块内自由使用。 上面代码中，`m()`方法不是泛型方法，`m1()`与`m2()`都是。值得注意的是`m2()`方法中声明的类型`T`与类申明里面的那个参数`T`不是一个，也可以说方法中的`T`**隐藏**了类型中的`T`。下面代码中类里面的`T`传入的是`String`类型，而方法中的`T`传入的是`Integer`类型。

```
Generic<String> str=new Generic<>("总有刁民想害朕");
str.m2(123);
```

###  泛型方法
在java中,泛型类的定义非常简单，但是泛型方法就比较复杂了。

尤其是我们见到的大多数泛型类中的成员方法也都使用了泛型，有的甚至泛型类中也包含着泛型方法，这样在初学者中非常容易将泛型方法理解错了。

泛型类，是在实例化类的时候指明泛型的具体类型；泛型方法，是在调用方法的时候指明泛型的具体类型 。
```java
/**
 * 泛型方法的基本介绍
 * @param tClass 传入的泛型实参
 * @return T 返回值为T类型
 * 说明：
 *     1）public 与 返回值中间<T>非常重要，可以理解为声明此方法为泛型方法。
 *     2）只有声明了<T>的方法才是泛型方法，泛型类中的使用了泛型的成员方法并不是泛型方法。
 *     3）<T>表明该方法将使用泛型类型T，此时才可以在方法中使用泛型类型T。
 *     4）与泛型类的定义一样，此处T可以随便写为任意标识，常见的如T、E、K、V等形式的参数常用于表示泛型。
 */
public <T> T genericMethod(Class<T> tClass)throws InstantiationException ,
  IllegalAccessException{
        T instance = tClass.newInstance();
        return instance;
}
```

```java
Object obj = genericMethod(Class.forName("com.test.test"));
```

### 泛型方法的基本用法
```java
public class GenericTest {
   //这个类是个泛型类，在上面已经介绍过
   public class Generic<T>{     
        private T key;

        public Generic(T key) {
            this.key = key;
        }

        //我想说的其实是这个，虽然在方法中使用了泛型，但是这并不是一个泛型方法。
        //这只是类中一个普通的成员方法，只不过他的返回值是在声明泛型类已经声明过的泛型。
        //所以在这个方法中才可以继续使用 T 这个泛型。
        public T getKey(){
            return key;
        }

        /**
         * 这个方法显然是有问题的，在编译器会给我们提示这样的错误信息"cannot reslove symbol E"
         * 因为在类的声明中并未声明泛型E，所以在使用E做形参和返回值类型时，编译器会无法识别。
        public E setKey(E key){
             this.key = keu
        }
        */
    }

    /** 
     * 这才是一个真正的泛型方法。
     * 首先在public与返回值之间的<T>必不可少，这表明这是一个泛型方法，并且声明了一个泛型T
     * 这个T可以出现在这个泛型方法的任意位置.
     * 泛型的数量也可以为任意多个 
     *    如：public <T,K> K showKeyName(Generic<T> container){
     *        ...
     *        }
     */
    public <T> T showKeyName(Generic<T> container){
        System.out.println("container key :" + container.getKey());
        //当然这个例子举的不太合适，只是为了说明泛型方法的特性。
        T test = container.getKey();
        return test;
    }

    //这也不是一个泛型方法，这就是一个普通的方法，只是使用了Generic<Number>这个泛型类做形参而已。
    public void showKeyValue1(Generic<Number> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

    //这也不是一个泛型方法，这也是一个普通的方法，只不过使用了泛型通配符?
    //同时这也印证了泛型通配符章节所描述的，?是一种类型实参，可以看做为Number等所有类的父类
    public void showKeyValue2(Generic<?> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

     /**
     * 这个方法是有问题的，编译器会为我们提示错误信息："UnKnown class 'E' "
     * 虽然我们声明了<T>,也表明了这是一个可以处理泛型的类型的泛型方法。
     * 但是只声明了泛型类型T，并未声明泛型类型E，因此编译器并不知道该如何处理E这个类型。
    public <T> T showKeyName(Generic<E> container){
        ...
    }  
    */

    /**
     * 这个方法也是有问题的，编译器会为我们提示错误信息："UnKnown class 'T' "
     * 对于编译器来说T这个类型并未项目中声明过，因此编译也不知道该如何编译这个类。
     * 所以这也不是一个正确的泛型方法声明。
    public void showkey(T genericObj){

    }
    */

    public static void main(String[] args) {


    }
}
```

### 类中的泛型方法
当然这并不是泛型方法的全部，泛型方法可以出现杂任何地方和任何场景中使用。但是有一种情况是非常特殊的，当泛型方法出现在泛型类中时，我们再通过一个例子看一下
```java
public class GenericFruit {
    class Fruit{
        @Override
        public String toString() {
            return "fruit";
        }
    }

    class Apple extends Fruit{
        @Override
        public String toString() {
            return "apple";
        }
    }

    class Person{
        @Override
        public String toString() {
            return "Person";
        }
    }

    class GenerateTest<T>{
        public void show_1(T t){
            System.out.println(t.toString());
        }

        //在泛型类中声明了一个泛型方法，使用泛型E，这种泛型E可以为任意类型。可以类型与T相同，也可以不同。
        //由于泛型方法在声明的时候会声明泛型<E>，因此即使在泛型类中并未声明泛型，编译器也能够正确识别泛型方法中识别的泛型。
        public <E> void show_3(E t){
            System.out.println(t.toString());
        }

        //在泛型类中声明了一个泛型方法，使用泛型T，注意这个T是一种全新的类型，可以与泛型类中声明的T不是同一种类型。
        public <T> void show_2(T t){
            System.out.println(t.toString());
        }
    }

    public static void main(String[] args) {
        Apple apple = new Apple();
        Person person = new Person();

        GenerateTest<Fruit> generateTest = new GenerateTest<Fruit>();
        //apple是Fruit的子类，所以这里可以
        generateTest.show_1(apple);
        //编译器会报错，因为泛型类型实参指定的是Fruit，而传入的实参类是Person
        //generateTest.show_1(person);

        //使用这两个方法都可以成功
        generateTest.show_2(apple);
        generateTest.show_2(person);

        //使用这两个方法也都可以成功
        generateTest.show_3(apple);
        generateTest.show_3(person);
    }
}
```

### 泛型方法与可变参数
再看一个泛型方法和可变参数的例子：
```java
public <T> void printMsg( T... args){
    for(T t : args){
        Log.d("泛型测试","t is " + t);
    }
}
```

```java
printMsg("111",222,"aaaa","2323.4",55.55);
```

###  静态方法与泛型
静态方法有一种情况需要注意一下，那就是在类中的静态方法使用泛型：静态方法无法访问类上定义的泛型；如果静态方法操作的引用数据类型不确定的时候，必须要将泛型定义在方法上。

即：如果静态方法要使用泛型的话，必须将静态方法也定义成泛型方法 。
```java
public class StaticGenerator<T> {
    ....
    ....
    /**
     * 如果在类中定义使用泛型的静态方法，需要添加额外的泛型声明（将这个方法定义成泛型方法）
     * 即使静态方法要使用泛型类中已经声明过的泛型也不可以。
     * 如：public static void show(T t){..},此时编译器会提示错误信息：
          "StaticGenerator cannot be refrenced from static context"
     */
    public static <T> void show(T t){

    }
}
```

### 泛型方法总结
泛型方法能使方法独立于类而产生变化，以下是一个基本的指导原则：

```
无论何时，如果你能做到，你就该尽量使用泛型方法。也就是说，如果使用泛型方法将整个类泛型化，  
  
那么就应该使用泛型方法。另外对于一个static的方法而已，无法访问泛型类型的参数。  
  
所以如果static方法要使用泛型能力，就必须使其成为泛型方法。
```


# 泛型的使用方法

## 如何继承一个泛型类

如果不传入具体的类型，则子类也需要指定类型参数，代码如下：

```text
class Son<T> extends Generic<T>{}
```

如果传入具体参数，则子类不需要指定类型参数

```text
class Son extends Generic<String>{}
```

## 如何实现一个泛型接口

```text
class ImageGenerator<T> implements Generator<T>{
    @Override
    public T produce() {
        return null;
    }
}
```

## 如何调用一个泛型方法

和调用普通方法一致，不论是实例方法还是静态方法。

### 通配符？

？代表任意类型，例如有如下函数：

```text
public void m3(List<?>list){
    for (Object o : list) {
        System.out.println(o);
    }
}
```

其参数类型是`？`，那么我们调用的时候就可以传入任意类型的`List`,如下

```text
str.m3(Arrays.asList(1,2,3));
str.m3(Arrays.asList("总有刁民","想害","朕"));
```

但是说实话，单独一个`？`意义不大，因为大家可以看到，从集合中获取到的对象的类型是`Object` 类型的，也就只有那几个默认方法可调用，几乎没什么用。如果你想要使用传入的类型那就需要**强制类型转换**，这是我们接受不了的，不然使用泛型干毛。其真正强大之处是可以通过设置其上下限达到类型的灵活使用，且看下面分解**重点内容**。

### 通配符上界

通配符上界使用`<? extends T>`的格式，意思是需要一个**T类型**或者**T类型的子类**，一般T类型都是一个具体的类型，例如下面的代码。

```text
public void printIntValue(List<? extends Number> list) {  
    for (Number number : list) {  
        System.out.print(number.intValue()+" ");   
    }  
}
```

这个意义就非凡了，无论传入的是何种类型的集合，我们都可以使用其父类的方法统一处理。

### 通配符下界

通配符下界使用`<? super T>`的格式，意思是需要一个**T类型**或者**T类型的父类**，一般T类型都是一个具体的类型，例如下面的代码。

```text
public void fillNumberList(List<? super Number> list) {  
    list.add(new Integer(0));  
    list.add(new Float(1.0));  
}
```

至于什么时候使用通配符上界，什么时候使用下界，在**《Effective Java》**中有很好的指导意见：遵循**PECS**原则，即**producer-extends,consumer-super.** 换句话说，如果参数化类型表示一个生产者，就使用 `<? extends T>`；如果参数化类型表示一个消费者，就使用`<? super T>`。

## 泛型在静态方法中的问题

**泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数**，例如下面的代码编译失败

```text
public class Test<T> {      
    public static T one;   //编译错误      
    public static  T show(T one){ //编译错误      
        return null;      
    }      
}
```

因为静态方法和静态变量属于类所有，而**泛型类中的泛型参数的实例化是在创建泛型类型对象时指定的，所以如果不创建对象，根本无法确定参数类型**。但是**静态泛型方法**是可以使用的，我们前面说过，泛型方法里面的那个类型和泛型类那个类型完全是两回事。

```text
public static <T>T show(T one){   
     return null;      
 }
```

# Java泛型原理解析

为什么人们会说Java的泛型是**伪泛型**呢，就是因为Java在编译时**擦除**了所有的泛型信息，所以Java根本不会产生新的类型到字节码或者机器码中，所有的泛型类型最终都将是一种**原始类型**，那样在Java运行时根本就获取不到泛型信息。


## 擦除

Java编译器编译泛型的步骤： 
* 1 **检查**泛型的类型 ，获得目标类型 
* 2 **擦除**类型变量，并替换为限定类型（T为无限定的类型变量，用Object替换） 
* 3 调用相关函数，并将结果**强制转换**为目标类型。

```text
ArrayList<String> arrayString=new ArrayList<String>();     

ArrayList<Integer> arrayInteger=new ArrayList<Integer>();     

System.out.println(arrayString.getClass()==arrayInteger.getClass());
```

上面代码输入结果为 **true**，可见通过运行时获取的类信息是完全一致的，泛型类型被擦除了！

**如何擦除：** 当擦除泛型类型后，留下的就只有**原始类型**了，例如上面的代码，原始类型就是`ArrayList`。擦除类型变量，并替换为限定类型（T为无限定的类型变量，用Object替换），如下所示

擦除之前：

```text
//泛型类型  
class Pair<T> {    
    private T value;    
    public T getValue() {    
        return value;    
    }    
    public void setValue(T  value) {    
        this.value = value;    
    }    
}
```

擦除之后：

```text
//原始类型  
class Pair {    
    private Object value;    
    public Object getValue() {    
        return value;    
    }    
    public void setValue(Object  value) {    
        this.value = value;    
    }    
}
```

因为在`Pair<T>`中，`T`是一个无限定的类型变量，所以用Object替换。如果是`Pair<T extends Number>`，擦除后，类型变量用Number类型替换。

## 与其他语言相比较

相比较**Java**,其模仿者**C#**在泛型方面无疑做的更好，其是真泛型。

C#泛型类在编译时，先生成中间代码`IL`，通用类型`T`只是一个占位符。在实例化类时，根据用户指定的数据类型代替T并由即时编译器（JIT）生成本地代码，这个本地代码中已经使用了实际的数据类型，等同于用实际类型写的类，所以不同的封闭类的本地代码是不一样的。其可以在运行时通过反射获得泛型信息，而且C#的泛型大大提高了代码的执行效率。

那么Java为什么不采用类似C#的实现方式呢，答案是：要向下兼容！兼容害死人啊。关于C#与Java的泛型比较，可以查看这篇文章：[Comparing Java and C# Generics](https://link.zhihu.com/?target=http%3A//www.jprl.com/Blog/archive/development/2007/Aug-31.html)




# 参考
https://zhuanlan.zhihu.com/p/64585072
https://juejin.cn/post/6844903650788114439
https://www.cnblogs.com/jingmoxukong/p/12049160.html