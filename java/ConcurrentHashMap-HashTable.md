一小时搞定HashMap & 
ConcurrentHashMap& HashTable面试
1）HashMap原理解析：扩容，阈值设置，死循环，红黑树
2）HashMap&HashTable面试问题总结
3）ConcurrentHashMap原理5mins 搞定；
4）ConcurrentHashMap分段锁&性能优化
5）如何在准备面试数据结构算法刷题


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303162031047.png)


问题1：可以序列化，通过 readObject 和 writeObject

hashcode：位运算，算出来类似求模
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303162038167.png)


问题2：扩容原理；超过填充因子0.75，阈值；为什么是2的幂次，提升效率

ArrayList 扩容


问题3：

