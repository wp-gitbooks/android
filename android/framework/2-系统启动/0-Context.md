# 参考

http://gityuan.com/2017/04/09/android_context/


https://duanqz.github.io/2017-12-25-Android-Context#11-%E9%9D%A2%E5%90%91%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E7%9A%84%E8%AE%BE%E8%AE%A1


# 概述

![context](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210426094805.jpg)


```plantuml
class ContextWrapper extends Context {
	Context mBase;
}

```
mBase 就是 ContextImpl


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303162243567.png)


# Context创建过程

http://liuwangshu.cn/framework/context/2-activity-service.html




![VejzTg.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210530134754.png)



# 面试题



http://gityuan.com/2017/04/09/android_context/

## Android应用中哪些是 Context，一个应用有多少个 Context？

https://www.jianshu.com/p/cc0bb2a71ee8

Context数量 = Activity数量 + Service数量 + 1 （1为Application）

## 讲解一下Context 

Context是一个抽象基类。在翻译为上下文，也可以理解为环境，是提供一些程序的运行环境基础信息。Context下有两个子类，ContextWrapper是上下文功能的封装类，而ContextImpl则是上下文功能的实现类。而ContextWrapper又有三个直接的子类， ContextThemeWrapper、Service和Application。其中，ContextThemeWrapper是一个带主题的封装类，而它有一个直接子类就是Activity，所以Activity和Service以及Application的Context是不一样的，只有Activity需要主题，Service不需要主题。

Context一共有三种类型，分别是Application、Activity和Service。这三个类虽然分别各种承担着不同的作用，但它们都属于Context的一种，而它们具体Context的功能则是由ContextImpl类去实现的，因此在绝大多数场景下，Activity、Service和Application这三种类型的Context都是可以通用的。不过有几种场景比较特殊，比如启动Activity，还有弹出Dialog。出于安全原因的考虑，Android是不允许Activity或Dialog凭空出现的，一个Activity的启动必须要建立在另一个Activity的基础之上，也就是以此形成的返回栈。而Dialog则必须在一个Activity上面弹出（除非是System Alert类型的Dialog），因此在这种场景下，我们只能使用Activity类型的Context，否则将会出错。



getApplicationContext()和getApplication()方法得到的对象都是同一个application对象，只是对象的类型不一样。

Context数量 = Activity数量 + Service数量 + 1 （1为Application）



## 为什么Dialog不能用Application的Context？

https://blog.csdn.net/jia635/article/details/52387658

https://juejin.cn/post/6844904115105955853





## Context 的使用上如何避免内存泄漏？





