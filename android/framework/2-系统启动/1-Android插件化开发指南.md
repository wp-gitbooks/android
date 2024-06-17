# Binder原理

![image-20210723104519826](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723104519.png)





![image-20210723104547878](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723104547.png)



# AIDL原理

![image-20210723104911593](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723104911.png)





![image-20210723104854256](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723104854.png)


# AMS
[[0-系统启动]]

## Activity启动

![image-20210723110829726](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723110829.png)


### 流程

![image-20210723110449991](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723110450.png)


#### 第一阶段：Launcher通知AMS

##### 第一步和第二步



![image-20210723110549478](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723110549.png)


##### 第三步

![image-20210723110625963](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723110626.png)



##### 第四步

![image-20210723110727493](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723110727.png)



##### 第五步

![image-20210723110757680](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723110757.png)
#### 第二阶段

![image-20210723111056541](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111056.png)



#### 第三阶段
![image-20210723111127936](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111127.png)



#### 第四阶段

![image-20210723111203035](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111203.png)



#### 第五阶段
![image-20210723111236041](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111236.png)



#### 第六阶段

![image-20210723111303945](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111303.png)



#### 第七阶段

![image-20210723111335784](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111335.png)



## Activity内部跳转
![image-20210723111420377](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111420.png)

# ClassLoader



![image-20210723111849509](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111849.png)


## 6.说说你对类加载机制的了解？DexClassLoader与PathClassLoader的区别



# hook

## 对AMN的hook

![image-20210723112142806](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723112142.png)



## 对PMS的hook

![image-20210723112227918](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723112227.png)



# 第5章 对startActivity方法进行Hook
## 1 startActivity方法的两种启动形式
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801094655.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801094725.png)

## 对Activity的startActivity方法进行Hook
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801094815.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801094856.png)

### hook点
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801094927.png)


#### 方案1：重写Activity的startActivityForResult方法
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095035.png)


#### 方案2：对Activity的mInstrumentation 字段进行Hook
这种Hook的方式有一个很大的缺点-----只针对当前Activity生效，因为它只修改了当前Activity实例的mInstrumentation字段

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095123.png)


#### 方案3：对AMN的getDefault方法进行Hook
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095320.png)


#### 方案4：对H类的mCallback字段进行Hook
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095406.png)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095438.png)


#### 对比
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095519.png)


#### 方案5：再次对Instrumentation字段进行Hook
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095602.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095621.png)


## 对Context的startActivity方法进行Hook
### 方案6：对ActivityThread的mInstrumentation字段进行Hook
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095804.png)


### 对AMN的getDefault方法进行Hook是一劳永逸的
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210801095845.png)



## 启动没有在AndroidManifest中声明的Activity



# Service工作原理



## 生命周期

![image-20210802210643949](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802210644.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210805162211.png)


## 启动

### 在新进程启动Service

![image-20210802210725999](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802210726.png)



#### 第1阶段：App向AMS发送一个启动Service的消息



![image-20210802210801839](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802210801.png)



#### 第2阶段：AMS创建新的进程

![image-20210802210829818](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802210829.png)





#### 第三阶段：新进程启动后，通知AMS，"我可以拉"

![image-20210802210908756](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802210908.png)



#### 第四阶段：AMS把刚才保存的Service信息发送给新进程

![image-20210802210955620](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802210955.png)



#### 第五阶段：新进程启动Service



![image-20210802211027151](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211027.png)



### 启动同一进程的Service

![image-20210802211107557](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211107.png)



### 在同一进程绑定Service

![image-20210802211234074](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211234.png)

#### 第1阶段：App向AMS发送一个绑定Service的消息

![image-20210802211313870](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211313.png)



#### 第2阶段：AMS创建新的进程

![image-20210802211337056](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211337.png)



#### 第3阶段：新进程启动后，通知AMS，"我可以啦"

![image-20210802211410421](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211410.png)



#### 第4阶段：处理第2个消息，绑定Service

![image-20210802211436600](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211436.png)



#### 第5阶段：把Binder对象发送给App

![image-20210802211511057](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211511.png)







# PMS及App安装过程 [[3-PMS]]
[[Hook机制之AMS&PMS]]



![image-20210802211642602](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211642.png)



## App的安装流程  [[3-PMS#启动]]

![image-20210802211725580](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211725.png)



## PackageParser

![image-20210802211824927](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802211825.png)



## ActivityThread与PackageManager

![image-20210802212022434](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802212022.png)





# ClassLoader家族史  [[技术/Android/framework/热修复#反射与类加载]]

![image-20210802212142084](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802212142.png)


```plantuml
class DexPathList {
	Element[] dexElements;
	Element[] nativeLibraryPathElements;
}
```


## 线索
![image-20210723113902411](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723113902.png)



## 双亲委托

![image-20210802212248151](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210802212248.png)



# so的插件化  [[Sytem.loadLibrary解析]]
http://gityuan.com/2017/03/26/load_library/
方案一：1、通过System.load直接加载路径
方案二：2、先通过DexClassLoader加载so的路径，然后再通过System.loadLibrary加载so
方案三：3、通过nativeLibraryPathElements
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210810210053.png)



## 加载so的两种方法
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210803144427.png)



![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210803150029.png)

## ClassLoader与so的关系
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210803144547.png)



## 基于System.load的插件化解决方案

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210803144642.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210803144728.png)


## 基于System.loadLibrary的插件化解决方案

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210803150508.png)

