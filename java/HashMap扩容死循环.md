# 产生循环链表的过程:

如下所示的hashmap, 有两个元素, 它们的key分别是1和3, 假设再增加一个元素时会触发扩容操作



![在这里插入图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210722111430.png)



此时线程1和线程2都执行put()操作, 便都会触发hashmap的扩容操作,

假设线程1扩容时, 执行完transfer()中的`Entry<K,V> next = e.next;`被挂起, 此时e指向1, next指向3, 如下图所示

![在这里插入图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210722111443.png)



假设1和3在新的数组中仍然发生哈希碰撞, 假设线程2完成了扩容, 那么此时哈希表的样子如下图所示
可以发现, 由于使用了头插法, 所以3变成了头结点

![在这里插入图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210722111513.png)



回到线程1, e指向的是1, next指向的是3, 继续向下执行
当执行完`e.next = newTable[i];`便出现了循环链表, 其中,`newTable[i]`是3

![在这里插入图片描述](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210722111532.png)

# 产生循环链表后带来的问题是什么?

环形链表已经产生了, 当我们调用get(3)或者get(1)不会产生问题,
但是如果get(5), 并且5在数组中的下标和1,3的一致的话, 由于链表中没有5, 所以就会一直在链表中寻找, 但是链表没有尽头, 就导致程序卡在get(5)处了





# 参考

https://blog.csdn.net/littlehaes/article/details/105241194

https://coolshell.cn/articles/9606.html

https://www.toutiao.com/i6800714048401834500/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1617232822&app=news_article&utm_source=weixin&utm_medium=toutiao_ios&use_new_style=1&req_id=202104010720210102120702050A24F90A&share_token=983D5780-133A-45B2-9910-2F01E98765AA&group_id=6800714048401834500

