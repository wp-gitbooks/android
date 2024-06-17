# 概述

## View

### View是什么

- View类是Android中各种组件的基类，如View是ViewGroup基类（Android中的UI组件都由View、ViewGroup组成）
- View表现为显示在屏幕上的各种视图

![image-20210324092119178](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324092119.png)

### view的构造函数

View的构造函数：共有4个（自定义View必须重写至少一个构造函数）

```
// 如果View是在Java代码里面new的，则调用第一个构造函数
 public CarsonView(Context context) {
        super(context);
    }

// 如果View是在.xml里声明的，则调用第二个构造函数
// 自定义属性是从AttributeSet参数传进来的
    public  CarsonView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

// 不会自动调用
// 一般是在第二个构造函数里主动调用
// 如View有style属性时
    public  CarsonView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    //API21之后才使用
    // 不会自动调用
    // 一般是在第二个构造函数里主动调用
    // 如View有style属性时
    public  CarsonView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }
```



详解：

http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0806/4575.html

https://www.cnblogs.com/angeldevil/p/3479431.html#three



### View的分类

视图View主要分为两类：

| 类别     | 解释                                      | 特点         |
| -------- | ----------------------------------------- | ------------ |
| 单一视图 | 即一个View，如TextView                    | 不包含子View |
| 视图组   | 即多个View组成的ViewGroup，如LinearLayout | 包含子View   |



## ViewGroup是什么



## View视图结构

![View树结构](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324093046.png)



## Android坐标系

Android的坐标系定义为：

- 屏幕的左上角为坐标原点
- 向右为x轴增大方向
- 向下为y轴增大方向

具体如下图：

![屏幕坐标系](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324093215.png)

![两者坐标系的区别](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324093232.png)



### View位置（坐标）描述

![View的顶点](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324093304.png)

4个顶点的位置描述分别由4个值决定：
（请记住：View的位置是相对于父控件而言的）

Top：子View上边界到父view上边界的距离
Left：子View左边界到父view左边界的距离
Bottom：子View下边距到父View上边界的距离
Right：子View右边界到父view左边界的距离
![View的位置描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324093329.png)



### 位置获取方式

- View的位置是通过`view.getxxx()`函数进行获取：**（以Top为例）**

```
// 获取Top位置
public final int getTop() {  
    return mTop;  
}  

// 其余如下：
  getLeft();      //获取子View左上角距父View左侧的距离
  getBottom();    //获取子View右下角距父View顶部的距离
  getRight();     //获取子View右下角距父View左侧的距离

```

- 与MotionEvent中 `get()`和`getRaw()`的区别

```
  //get() ：触摸点相对于其所在组件坐标系的坐标
   event.getX();       
   event.getY();
  
  //getRaw() ：触摸点相对于屏幕默认坐标系的坐标
   event.getRawX();    
   event.getRawY();
```

  ![get() 和 getRaw() 的区别](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324093507.png)

### Android的角度(angle)与弧度(radian)







## View滑动

scrollTo和scrollBy

《Android开发艺术探索》



### 横向 ScrollView、纵向 ListView 怎么处理滑动手势冲突

https://blog.csdn.net/xiaohanluo/article/details/52130923







## 事件分类

### MotionEvent事件动作序列

一次完整的MotionEvent事件，是从用户触摸屏幕到离开屏幕。整个过程的动作序列：ACTION_DOWN(1次) -> ACTION_MOVE(N次) -> ACTION_UP(1次)



# View、ViewGroup、Activity之间关系

Activity

​		PhoneWindow

​				DecorView---------->FrameLayout------>ViewGroup

​						View



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210308152839)

# ViewRoot、DecorView、Window之间关系

## ViewRoot

![image-20210419152233189](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210419152233.png)



```
// 在主线程中，Activity对象被创建后：
// 1. 自动将DecorView添加到Window中 & 创建ViewRootImpll对象
root = new ViewRootImpl(view.getContent(),display);

// 3. 将ViewRootImpll对象与DecorView建立关联
root.setView(view,wparams,panelParentView)
```

## DecorView

### 定义

顶层View，即 Android 视图树的根节点；同时也是 FrameLayout 的子类

### 作用

显示 & 加载布局。View层的事件都先经过DecorView，再传递到View

### 组成

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210308220929.png)

```
// 在代码中可通过content得到对应加载的布局

// 1. 得到content
ViewGroup content = (ViewGroup)findViewById(android.R.id.content);
// 2. 得到设置的View
ViewGroup rootView = (ViewGroup) content.getChildAt(0);

```



## Window

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210308221252.png)

## Activity

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210308221316.png)

### 关系

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210308221353.png)



![在这里插入图片描述](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210308221414.png)
# 问题

## 简述Activity与Window关系

http://gityuan.com/2017/04/16/activity-with-window/



## Activity,view,window联系（理解Activity，View,Window三者关系）

这个问题真的很不好回答。所以这里先来个算是比较恰当的比喻来形容下它们的关系吧。Activity像一个工匠（控制单元），Window像窗户（承载模型），View像窗花（显示视图）LayoutInflater像剪刀，Xml配置像窗花图纸。

1：Activity构造的时候会初始化一个Window，准确的说是PhoneWindow。

2：这个PhoneWindow有一个“ViewRoot”，这个“ViewRoot”是一个View或者说ViewGroup，是最初始的根视图。

3：“ViewRoot”通过addView方法来一个个的添加View。比如TextView，Button等

4：这些View的事件监听，是由WindowManagerService来接受消息，并且回调Activity函数。比如onClickListener，onKeyDown等。





## DecorView, ViewRootImpl,View之间的关系，ViewGroup.add()会多添加一个ViewrootImpl吗




## 如何通过WindowManager添加Window(代码实现)？



## 请解释一下DecorView，它有什么作用？

**参考回答：** 有下面一个视图，DecorView为整个Window界面的最顶层View，它只有一个子元素LinearLayout。代表整个Window界面，包含通知栏、标题栏、内容显示栏三块区域。其中LinearLayout中有两个FrameLayout子元素。

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210302104514)



![image-20210302104444687](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210302104444.png)



**DecorView的作用**：DecorView是顶级View，本质是一个FrameLayout它包含两部分，标题栏和内容栏，都是FrameLayout。内容栏id是content，也就是activity中设置setContentView的部分，最终将布局添加到id为content的FrameLayout中。
 获取content：ViewGroup content=findViewById（android.id.content）
 获取设置的View：getChildAt(0).

####

## RecyclerView与ListView的对比，缓存策略，优缺点



## View的滑动方式



## Activity,Window,View三者的联系和区别



##  ListView卡顿的原因以及优化策略



## ViewHolder为什么要被声明成静态内部类



## RecyclerView是什么？如何使用？如何返回不一样的Item

## RecyclerView的回收复用机制



## 如何给ListView & RecyclerView加上拉刷新 & 下拉加载更多机制



## 如何对ListView & RecycleView进行局部刷新的？



## ScrollView下嵌套一个RecycleView通常会出现什么问题？



##一个ListView或者一个RecyclerView在显示新闻数据的时候，出现图片错位，可能的原因有哪些 & 如何解决？

