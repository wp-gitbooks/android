# Class.forName 和 ClassLoader 有什么区别？
根据运行结果得出 **Class.forName 加载类是将类进了初始化，而 ClassLoader 的 loadClass 并没有对类进行初始化，只是把类加载到了虚拟机中** 。


在 java 中 Class.forName() 和 ClassLoader 都可以对类进行加载。 ClassLoader 就是遵循双亲委派模型最终调用启动类加载器的类加载器，实现的功能是“通过一个类的全限定名来获取描述此类的二进制字节流”，获取到二进制流后放到 JVM 中。 Class.forName() 方法实际上也是调用的 ClassLoader 来实现的。

Class.forName(String className)； 这个方法的源码是：

![新知达人, 面试题：Class.forName 和 ClassLoader 有什么区别？](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210807221911.png)

最后调用的方法是 forName0 这个方法，在这个 forName0 方法中的第二个参数被默认设置为了 true，这个参数代表是否对加载的类进行初始化，设置为 true 时会类进行初始化，代表会执行类中的静态代码块，以及对静态变量的赋值等操作。

也可以调用 Class.forName(String name, boolean initialize,ClassLoader loader) 方法来手动选择在加载类的时候是否要对类进行初始化。Class.forName(String name, boolean initialize,ClassLoader loader) 的源码如下：

![新知达人, 面试题：Class.forName 和 ClassLoader 有什么区别？](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210807221920.jpeg)

源码中的注释只摘取了一部分，其中对参数 initialize 的描述是：if {@code true} the class will be initialized. 意思就是说：如果参数为 true，则加载的类将会被初始化。

## 举例:
下面还是举例来说明结果吧：一个含有静态代码块、静态变量、赋值给静态变量的静态方法的类。

![新知达人, 面试题：Class.forName 和 ClassLoader 有什么区别？](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210807221926.png)

测试方法：

![新知达人, 面试题：Class.forName 和 ClassLoader 有什么区别？](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210807221939.jpeg)

运行结果：

![新知达人, 面试题：Class.forName 和 ClassLoader 有什么区别？](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210807221945.png)

根据运行结果得出 **Class.forName 加载类是将类进了初始化，而 ClassLoader 的 loadClass 并没有对类进行初始化，只是把类加载到了虚拟机中** 。

## 应用场景

在我们熟悉的 Spring 框架中的 IOC 的实现就是使用的 ClassLoader。

而在我们使用 JDBC 时通常是使用 Class.forName() 方法来加载数据库连接驱动。这是因为在 JDBC 规范中明确要求 Driver(数据库驱动)类必须向 DriverManager 注册自己。

以 MySQL 的驱动为例解释：

![新知达人, 面试题：Class.forName 和 ClassLoader 有什么区别？](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210807221956.jpeg)

我们看到 Driver 注册到 DriverManager 中的操作写在了静态代码块中，这就是为什么在写 JDBC 时使用 Class.forName() 的原因。