# 线索

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303072120825.png)

# SP实现
IO -> 读取文件数据byte[] ->xml解析->ArrayMap中保存供使用
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303072129628.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303072153942.png)

# 数据结构
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303072154227.png)

# 跨进程通信
## 传统I/O
传统IO：经过两次数据拷贝，一次从用户空间到内核空间，一次从内核空间到文件
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303072158717.png)

## MMAP
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303072203311.png)


用户空间同文件直接映射，不需要经过内核，只需要一次拷贝。
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303072205609.png)

# MMAP使用
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081359528.png)

Linux采用了分页来管理内存，即内存的管理中，内存是以页为单位，一般的32系统一页为4096字节
size:一页倍数的大小 4096 字节
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081404279.png)

MMKV采用变长编码：protobuf  1~5个字节
定长编码：int 4字节

## 更新方式
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081414310.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081415969.png)
有空间写入这次的新数据->把数据编码成mmkv文件格式，全量的覆盖写入文件

扩容：扩容文件大小-》解除映射-》重新映射
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081417325.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081419016.png)

多进程：文件锁  CRC校验https://www.bilibili.com/video/BV1wy4y1k7UU?p=5&spm_id_from=pageDriver&vd_source=f089cad7f2e400622e91946932061cb9
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202303081421248.png)
https://www.bilibili.com/video/BV1wy4y1k7UU?p=5&spm_id_from=pageDriver&vd_source=f089cad7f2e400622e91946932061cb9

https://blog.csdn.net/ldld1717/article/details/125920473

https://juejin.cn/post/7078640657807441934#heading-8
