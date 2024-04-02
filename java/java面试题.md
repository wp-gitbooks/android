# 面试题

https://github.com/AobingJava/JavaFamily

























## java中中文字符和英文字符的大小分别多少？在网络上传输大小又分别是多少？













##下面的代码， str 值最终为多少？换成 Integer 值又为多少，是否会被改变？

1. 下面的代码， str 值最终为多少？换成 Integer 值又为多少，是否会被改变？

> - **考点**：Java 值传递 (第 2 题相同)。编写代码测试，在 changeValue() 方法中修改入参，并**不会改变**之前的值；
> - **原理** ：[Java 程序设计语言总是采用按值调用](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FSnailclimb%2FJavaGuide%2Fblob%2Fmaster%2Fdocs%2Fessential-content-for-interview%2FPreparingForInterview%2F%E5%BA%94%E5%B1%8A%E7%94%9F%E9%9D%A2%E8%AF%95%E6%9C%80%E7%88%B1%E9%97%AE%E7%9A%84%E5%87%A0%E9%81%93Java%E5%9F%BA%E7%A1%80%E9%97%AE%E9%A2%98.md%23%E4%B8%80-%E4%B8%BA%E4%BB%80%E4%B9%88-java-%E4%B8%AD%E5%8F%AA%E6%9C%89%E5%80%BC%E4%BC%A0%E9%80%92)，方法得到的是所有参数值的一个拷贝，即方法**不能修改**传递给它的任何参数变量的内容。基本类型参数传递的是参数副本，对象类型参数传递的是**对象地址的副本；**
> - **题解**：在 changeValue() 中，对于对象类型参数，直接修改的是**对象地址副本**的值，所以之前变量的地址并未被修改！若修改的是对象实例里面的某个值，之前变量则会被修改



```rust
public void test() {
    String str = "123";
    changeValue(str); 
    System.out.println("str值为: " + str);  // str未被改变，str = "123"
}

public changeValue(String str) {
    str = "abc";
}
```



作者：叛逆的青春不回头
链接：https://www.jianshu.com/p/5e5908ab3ea9
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



## 下面的代码，再次使用对象 student 是否需要判空？

1. 下面的代码，再次使用对象 student 是否需要判空？

> **Java 中方法参数的使用情况总结：**
>
> - 一个方法不能修改一个基本数据类型的参数（即数值型或布尔型）；
> - 一个方法可以改变一个对象参数的状态；
> - 一个方法不能让对象参数引用一个新的对象



```csharp
public void test()  {
    Student student = new Student("Bobo", 15);
    changeValue1(student);   // student值未改变，不为null! 输出结果 student值为 name:Bobo、age:15
    // changeValue2(student);  // student值被改变，输出结果 student值为 name:Lily、age:20
    System.out.println("student值为 name: " + student.name + "、age:" + student.age);
}

public changeValue1(Student student) {
    student = null;
}

public static void changeValue2(Student student)  {    
     student.name = "Lily";    
     student.age = 20;
}
```



