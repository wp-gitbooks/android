# 一、Comparable与Comparator的相同点

Comparable和Comparator都是java的一个接口，多用于实现集合中元素的比较及排序。
当我们自定义一个类时，如果需要规定其中的排序规则时，我们就必须用到比较接口。例如：

```arduino
public class Person{
    private String name;//姓名
    private int age;//年龄
    private double height;//身高
}
```

我们定义了一个类，想规定其按照年龄进行比较，并且将Person类的对象存在List集合中，调用集合工具类中的 ***Collection.sort\*** 排序方法，这时不能得到我们想要得到的结果。而我们 ***String\*** 却不然，如果我们讲 String 存入到List集合中，它会按照一定的顺序进行排序，因为我们String类中实现了 ***Comparable\*** 接口，并重写了接口中的 ***compareTo\*** 方法，如下：

```arduino
public int compareTo(String anotherString) {
    int len1 = value.length;
    int len2 = anotherString.value.length;
    int lim = Math.min(len1, len2);
    char v1[] = value;
    char v2[] = anotherString.value;
    int k = 0;
    while (k < lim) {
        char c1 = v1[k];
        char c2 = v2[k];
        if (c1 != c2) {
            return c1 - c2;
        }
        k++;
    }
    return len1 - len2;
}
```

所以我们需要给自己定义的类定义一个比较的规则，我们就需要实现一个比较接口。

# 二、Comparable 和 Comparator 的区别

![image](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210820175111.png)

## Comparable

### Comparable简介

Comparable是排序接口。若一个类实现了Comparable接口，就意味着该类支持排序。实现了Comparable接口的类的对象的列表或数组可以通过Collections.sort或Arrays.sort进行自动排序。

　　此外，实现此接口的对象可以用作有序映射中的键或有序集合中的集合，无需指定比较器。该接口定义如下：

```
package java.lang;
import java.util.*;
public interface Comparable<T> 
{
    public int compareTo(T o);
}
```



T表示可以与此对象进行比较的那些对象的类型。

此接口只有一个方法compare，比较此对象与指定对象的顺序，如果该对象小于、等于或大于指定对象，则分别返回负整数、零或正整数。



***Comparable\*** 定义在想要实现排序对象的类的内部，并且重写***compareTo\***方法，例如：

```angelscript
public class Person implements Comparable<Person>{
    private String name;//姓名
    private int age;//年龄
    private double height;//身高
    /**
     * 方法的作用就是定义比较规则
     * 返回值类型为int类型,取值范围分别为正数,负数,0
     * 正数:大于
     * 负数:小于
     * 0:等于
     * @param p
     * @return int
     */ 
     @Override
     public int compareTo(Person p) {
         return this.age - p.age;
     }
}
```

当一个类实现了***Comparable\***接口并重写了***CompareTo\***方法时，我们使用Collections.sort进行排序的话，会自动调用其自定义的比较规则进行比较排序，***CompareTo\***方法的返回值为int类型，有三种情况，分别为：

> 1、自定义的比较规则中，比较着大于被比较者时，则返回正数；
> 2、自定义的比较规则中，比较着小于被比较者时，则返回负数；
> 3、自定义的比较规则中，比较着等于被比较者时，则返回0；

## Comparator

### Comparator简介

　　Comparator是比较接口，我们如果需要控制某个类的次序，而该类本身不支持排序(即没有实现Comparable接口)，那么我们就可以建立一个“该类的比较器”来进行排序，这个“比较器”只需要实现Comparator接口即可。也就是说，我们可以通过实现Comparator来新建一个比较器，然后通过这个比较器对类进行排序。该接口定义如下：

```
package java.util;
public interface Comparator<T>
 {
    int compare(T o1, T o2);
    boolean equals(Object obj);
 }
```







***Comparator\*** 定义在想要实现排序对象的类的外部，并且重写***compare\***方法。
当一个类未定义比较规则，并且我们想要对其进行排序时，会使用你匿名内部类的方式或者重新自定义一个类并实现Comparator接口的方式来达到这个目的，例如：
**匿名内部类方式:**

```csharp
public class PersonTest {
    public static void main(String[] args) {
        Person p1 = new Person("张三", 18, 180);
        Person p2 = new Person("李四", 20, 165);
        Person p3 = new Person("王五", 15, 175);
        List<Person> list = new ArrayList();
        list.add(p1);
        list.add(p2);
        list.add(p3);
        Collections.sort(list, new Comparator<Person>() {
            @Override
            public int compare(Person o1, Person o2) {
                return o1.getAge()-o2.getAge();
            }
        });
        System.out.println(list);
 }
}
```

**自定义类实现Comparator接口**

```haxe
public class PersonComparator implements Comparator<Person> {
    @Override
    public int compare(Person o1, Person o2) {
        return o1.getAge()-o2.getAge();
    }
}
public class PersonTest {
    public static void main(String[] args) {
        Person p1 = new Person("张三", 18, 180);
        Person p2 = new Person("李四", 20, 165);
        Person p3 = new Person("王五", 15, 175);
        List<Person> list = new ArrayList();
        list.add(p1);
        list.add(p2);
        list.add(p3);
        Collections.sort(list, new PersonComparator());
        System.out.println(list);
 }
}
```

其实,从原理上来讲它们没有什么太大的不同，都是实现了 ***Comparator\*** 接口并重写了 ***Compare\*** 方法，只是写法上有些区别。当然从复用性的角度来讲，还是自定义的复用性更高一些，这里还需要实际看需求决定。它的比较规则和上述的Comparable中的CompareTo方法一样， ***Compare\*** 方法的返回值也为int类型，也有三种情况，分别为：

> 1、返回值类型为正数时,代表o1 > o2
> 2、返回值类型为负数时,代表o1 < o2
> 3、返回值类型为0时,代表o1 = o2

# 三、注意事项

无论是实现Comparable接口还是Comparator接口都需要指定**泛型**。

# 四、总结

实际上述两种方式各有利弊，实现Comparable使用比较简单，只要是实现了Comparable接口的类的对象，我们可以直接进行比较，但是在一些开发中我们开始未考虑到这里的情况下，想要改变就需要改变其源代码，不够方便；但是**Comparator**则不然，它可以在不修改源码的基础上，重新自定义一个比较器，这样**降低**了我们代码之间的**耦合性**，当我们之后想要改变一个比较算法时，也可以很方便的进行改变。我们只要重新定义一个比较规则即可。





# 参考

https://www.cnblogs.com/xujian2014/p/5215082.html