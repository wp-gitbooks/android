# 参考
http://gityuan.com/2016/03/12/start-activity/
https://www.cnblogs.com/andy-songwei/p/13508185.html


# 线索

![image-20210708154749870](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210708154749.png)



# Activity启动流程图[[1-Android插件化开发指南#AMS]]

![start_activity_process](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210310220039.jpg)



![start_activity](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210310220109.jpg)

## 类图
### ActivityManagerService
![activity_manager_classes](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210517111845.png)

### ApplicationThread

![application_thread_classes](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210517112122.png)
```plantuml
class ActivityThread {
	ContextImpl mSystemContext;
	ApplicationThread mAppThread;
	Looper mLooper;
	H mH;
	class ApplicationThread extends ApplicationThreadNative {
	}
}
```

```plantuml
class ApplicationThreadNative extends Binder implements IApplicationThread{        
	boolean onTransact(int code, Parcel data, Parcel reply, int flags);
	class ApplicationThreadProxy implements IApplicationThread {}
}
```


## 1 launcher进程到system_server进程
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303070919364.png)
## 2 AMS处理Launcher传过来的信息
AMS 处理Launcher 过来的 startActivity

## 3 给 Launcher 回调信息
![image-20210723111127936](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111127.png)

## 4 zygote启动新进程
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071004121.png)



## 5 新进程启动
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303070952928.png)

## 6 AMS告诉新App启动哪个Activity
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071006061.png)
## 7 启动新的Activity
![image-20210723111335784](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210723111335.png)



# 简述Activity生命周期

http://gityuan.com/2016/03/18/start-activity-cycle/

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303070939895.png)


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303070938277.png)
## Activity A 跳转到Activity B：
A执行onPause()，B执行onCreate()、onStart()、onResume() ，A再执行onStop()。
按返回键之后，B执行onPause()，A执行onStart()，onResume()，B在执行onStop()，onDestroy()。
