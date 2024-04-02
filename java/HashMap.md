# 构造方法
默认数组容量16，默认加载因子0.75
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210814124418.png)




# put
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210814124838.png)

## 头插法
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210814124231.png)




# 扩容
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210814174552.png)

## 循环链表-死循环





# 参考 [[4-大话数据结构-查找#散列表]]
https://github.com/cosen1024/Java-Interview/blob/main/Java%E9%9B%86%E5%90%88/HashMap.md

https://github.com/cosen1024/Java-Interview/blob/main/Java%E9%9B%86%E5%90%88/HashMap%E7%9A%84%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98.md



![image-20210415224352351](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210415224352.png)



https://www.bilibili.com/video/BV1kJ411C7hC?from=search&seid=9671694800014441838


JDK7中实现原理：数组+链表实现

JDK8中实现原理：数组+链表+红黑树实现





https://mp.weixin.qq.com/s/6DfOHiXxSEjxqxpfCGFtHA





https://www.toutiao.com/i6950580109808026125/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1618466481&app=news_article&utm_source=weixin&utm_medium=toutiao_ios&use_new_style=1&req_id=202104151401200102122052023E0783BB&share_token=DD40AB51-253E-41BC-B05A-5C61162300DE&group_id=6950580109808026125



https://tech.meituan.com/2016/06/24/java-hashmap.html



https://blog.csdn.net/pange1991/article/details/82347284





https://www.toutiao.com/i6800714048401834500/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1617232822&app=news_article&utm_source=weixin&utm_medium=toutiao_ios&use_new_style=1&req_id=202104010720210102120702050A24F90A&share_token=983D5780-133A-45B2-9910-2F01E98765AA&group_id=6800714048401834500



# HashMap,ConcurrentHashMap,HashTable有什么异同



# ConcurrentHashMap是不是绝对的线程安全。（final,绝对线程安全,相对线程安全,线程不安全）



# 为什么ConcurrentHashMap的读操作不需要加锁？

https://mp.weixin.qq.com/s/7_BVmQuPYnRbdifZGLiijw



# 面试题



## hashmap的实现



## weakhashmap

https://juejin.cn/post/6844903817859825672#heading-11



https://my.oschina.net/u/4405459/blog/4163941



## HashMap原理

## 为什么要链表转红黑树

## 为什么不一开始就用红黑树

## HashMap原理
        
## 为什么要链表转红黑树
        
## 为什么不一开始就用红黑树

## 美团面试题：hashCode 和对象的内存地址有什么关系？
https://mp.weixin.qq.com/s/teETH8Xw00TmE4ejqILGAQ

## java容器，hashmap和hashtable区别，hashmap原理，扩容流程，扰动算法的优势

## CurrentHashMap1.7和1.8区别

## hash算法，hashmap,怎么解决hash冲突

## hashmap原理,arraymap原理，对比性能

## hashmap为什么大于8才转化为红黑树，加载因子为什么是0.75

## Hashmap原理，给一个 key 如何计算槽位，如何取值?

## HashMap，HashTable，HashSet什么区别？

## Java中的HashSet内部是如何工作的


## 在使用 HashMap 的时候，用 String 做 key 有什么好处？

HashMap 内部实现是通过 key 的 hashcode 来确定 value 的存储位置，因为字符串是不可变的，所以当创建字符串时，它的 hashcode 被缓存下来，不需要再次计算，所以相比于其他对象更快。


## 用过哪些Map类，都有什么区别，HashMap是线程安全的吗,并发下使用的Map是什么，他们

内部原理分别是什么，比如存储方式，hashcode，扩容，默认容量等。

## JAVA8的ConcurrentHashMap为什么放弃了分段锁，有什么问题吗，如果你来设计，你如何


## HashMap底层实现，为什么扩容是2的幂次；
