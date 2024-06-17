---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 线索

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303150954468.png)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303061051329.png)



# 2 参考
https://juejin.cn/post/6904192359253147661#heading-6

https://cloud.tencent.com/developer/article/1601353

https://cloud.tencent.com/developer/article/1745688

https://developer.android.com/guide/topics/ui/how-android-draws?hl=zh-cn#layout

https://www.cnblogs.com/huansky/p/11911549.html

# 3 绘图流程

## 3.1 流程图

![image-20210311152838265](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210311152838.png)

## 3.2 函数调用链

![image-20210311153306195](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210311153306.png)


# 4 步骤

## 4.1 measure和layout
https://developer.android.com/guide/topics/ui/how-android-draws?hl=zh-cn#layout

![image-20210311153657526](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210311153657.png)

## 4.2 measure概述
了解`measure`过程前，需要先了解传递尺寸（宽 / 高测量值）的2个类：

- `ViewGroup.LayoutParams`类（）
- `MeasureSpecs` 类（父视图对子视图的测量要求）

### 4.2.1 简介
1. `ViewGroup` 的子类`（RelativeLayout、LinearLayout）`有其对应的 `ViewGroup.LayoutParams` 子类
2. 如：`RelativeLayout`的 `ViewGroup.LayoutParams`子类
   = `RelativeLayoutParams`

### 4.2.2 作用
- 指定视图`View` 的高度`（height）` 和 宽度`（width）`等布局参数。

### 4.2.3 具体使用

指定视图View 的高度（height） 和 宽度（width）等布局参数。

![image-20210324141347826](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324141348.png)


## 4.3 MeasureSpec

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324141427.png)

### 4.3.1 组成

测量规格`（MeasureSpec）` = 测量模式`（mode）` + 测量大小`（size）`

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324141616.png)

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324141656.png)


### 4.3.2 具体使用

- `MeasureSpec` 被封装在`View`类中的一个内部类里：`MeasureSpec`类
- MeasureSpec类 用1个变量封装了2个数据`（size，mode）`：**通过使用二进制**，将测量模式`（mode）` & 测量大小`(size）`打包成一个`int`值来，并提供了打包 & 解包的方法

> 该措施的目的 = 减少对象内存分配


### 4.3.3 实际使用

```java
/**
  * MeasureSpec类的具体使用
  **/

    // 1. 获取测量模式（Mode）
    int specMode = MeasureSpec.getMode(measureSpec)

    // 2. 获取测量大小（Size）
    int specSize = MeasureSpec.getSize(measureSpec)

    // 3. 通过Mode 和 Size 生成新的SpecMode
    int measureSpec=MeasureSpec.makeMeasureSpec(size, mode);
```

### 4.3.4 源码分析

```java
/**
  * MeasureSpec类的源码分析
  **/
    public class MeasureSpec {

        // 进位大小 = 2的30次方
        // int的大小为32位，所以进位30位 = 使用int的32和31位做标志位
        private static final int MODE_SHIFT = 30;  
          
        // 运算遮罩：0x3为16进制，10进制为3，二进制为11
        // 3向左进位30 = 11 00000000000(11后跟30个0)  
        // 作用：用1标注需要的值，0标注不要的值。因1与任何数做与运算都得任何数、0与任何数做与运算都得0
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;  
  
        // UNSPECIFIED的模式设置：0向左进位30 = 00后跟30个0，即00 00000000000
        // 通过高2位
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;  
        
        // EXACTLY的模式设置：1向左进位30 = 01后跟30个0 ，即01 00000000000
        public static final int EXACTLY = 1 << MODE_SHIFT;  

        // AT_MOST的模式设置：2向左进位30 = 10后跟30个0，即10 00000000000
        public static final int AT_MOST = 2 << MODE_SHIFT;  
  
        /**
          * makeMeasureSpec（）方法
          * 作用：根据提供的size和mode得到一个详细的测量结果吗，即measureSpec
          **/ 
            public static int makeMeasureSpec(int size, int mode) {  
            
                return size + mode;  
            // measureSpec = size + mode；此为二进制的加法 而不是十进制
            // 设计目的：使用一个32位的二进制数，其中：32和31位代表测量模式（mode）、后30位代表测量大小（size）
            // 例如size=100(4)，mode=AT_MOST，则measureSpec=100+10000...00=10000..00100  

            }  
      
        /**
          * getMode（）方法
          * 作用：通过measureSpec获得测量模式（mode）
          **/    

            public static int getMode(int measureSpec) {  
             
                return (measureSpec & MODE_MASK);  
                // 即：测量模式（mode） = measureSpec & MODE_MASK;  
                // MODE_MASK = 运算遮罩 = 11 00000000000(11后跟30个0)
                //原理：保留measureSpec的高2位（即测量模式）、使用0替换后30位
                // 例如10 00..00100 & 11 00..00(11后跟30个0) = 10 00..00(AT_MOST)，这样就得到了mode的值

            }  
        /**
          * getSize方法
          * 作用：通过measureSpec获得测量大小size
          **/       
            public static int getSize(int measureSpec) {  
             
                return (measureSpec & ~MODE_MASK);  
                // size = measureSpec & ~MODE_MASK;  
               // 原理类似上面，即 将MODE_MASK取反，也就是变成了00 111111(00后跟30个1)，将32,31替换成0也就是去掉mode，保留后30位的size  
            } 

    }  
```


### 4.3.5 MeasureSpec值的计算

子View的`MeasureSpec`值根据**子View的布局参数（LayoutParams）和父容器的MeasureSpec值**计算得来的，具体计算逻辑封装在`getChildMeasureSpec()`里

![对于普通View](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324143028.png)

#### 4.3.5.1 getChildMeasureSpec()的源码

```java
/**
  * 源码分析：getChildMeasureSpec（）
  * 作用：根据父视图的MeasureSpec & 布局参数LayoutParams，计算单个子View的MeasureSpec
  * 注：子view的大小由父view的MeasureSpec值 和 子view的LayoutParams属性 共同决定
  **/

    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {  

         //参数说明
         * @param spec 父view的详细测量值(MeasureSpec) 
         * @param padding view当前尺寸的的内边距和外边距(padding,margin) 
         * @param childDimension 子视图的布局参数（宽/高）

            //父view的测量模式
            int specMode = MeasureSpec.getMode(spec);     

            //父view的大小
            int specSize = MeasureSpec.getSize(spec);     
          
            //通过父view计算出的子view = 父大小-边距（父要求的大小，但子view不一定用这个值）   
            int size = Math.max(0, specSize - padding);  
          
            //子view想要的实际大小和模式（需要计算）  
            int resultSize = 0;  
            int resultMode = 0;  
          
            //通过父view的MeasureSpec和子view的LayoutParams确定子view的大小  


            // 当父view的模式为EXACITY时，父view强加给子view确切的值
           //一般是父view设置为match_parent或者固定值的ViewGroup 
            switch (specMode) {  
            case MeasureSpec.EXACTLY:  
                // 当子view的LayoutParams>0，即有确切的值  
                if (childDimension >= 0) {  
                    //子view大小为子自身所赋的值，模式大小为EXACTLY  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  

                // 当子view的LayoutParams为MATCH_PARENT时(-1)  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    //子view大小为父view大小，模式为EXACTLY  
                    resultSize = size;  
                    resultMode = MeasureSpec.EXACTLY;  

                // 当子view的LayoutParams为WRAP_CONTENT时(-2)      
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    //子view决定自己的大小，但最大不能超过父view，模式为AT_MOST  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                }  
                break;  
          
            // 当父view的模式为AT_MOST时，父view强加给子view一个最大的值。（一般是父view设置为wrap_content）  
            case MeasureSpec.AT_MOST:  
                // 道理同上  
                if (childDimension >= 0) {  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                }  
                break;  
          
            // 当父view的模式为UNSPECIFIED时，父容器不对view有任何限制，要多大给多大
            // 多见于ListView、GridView  
            case MeasureSpec.UNSPECIFIED:  
                if (childDimension >= 0) {  
                    // 子view大小为子自身所赋的值  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    // 因为父view为UNSPECIFIED，所以MATCH_PARENT的话子类大小为0  
                    resultSize = 0;  
                    resultMode = MeasureSpec.UNSPECIFIED;  
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    // 因为父view为UNSPECIFIED，所以WRAP_CONTENT的话子类大小为0  
                    resultSize = 0;  
                    resultMode = MeasureSpec.UNSPECIFIED;  
                }  
                break;  
            }  
            return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
        }  
```



![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324144122.png)


- 区别于顶级`View`（即`DecorView`）的测量规格`MeasureSpec`计算逻辑：取决于 **自身布局参数 & 窗口尺寸**

![对于顶级View](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324144206.png)



## 4.4 measure过程详解

https://blog.csdn.net/carson_ho/article/details/56011064

measure过程分为两类

![示意图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS01NTZiZjA5NGRmOTFiOWRlLnBuZz9pbWFnZU1vZ3IyL2F1dG8tb3JpZW50L3N0cmlwJTdDaW1hZ2VWaWV3Mi8yL3cvMTI0MA)



### 4.4.1 单一View的measure过程

> 实际作用的方法：`getDefaultSize()` = 计算View的宽/高值、`setMeasuredDimension（）` = 存储测量后的View宽 / 高

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324145652.png)

`getDefaultSize()`计算View的宽/高值的逻辑

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324145614.png)



### 4.4.2 ViewGroup的measure过程

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324145955.png)

原理

1. 遍历 测量所有子`View`的尺寸
2. 合并将所有子`View`的尺寸进行，最终得到`ViewGroup`父视图的测量值

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324145854.png)



- 流程

![示意图](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324145922.png)



## 4.5 View的invalidate和postInvalidate方法源码分析

https://blog.csdn.net/yanbober/article/details/46128379



## 4.6 View的requestLayout方法源码分析



# 5 setContentView源码分析
[[2-WMS]]

https://www.jianshu.com/p/1f99da8396b6

https://kknews.cc/code/gzoy48.html


![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071442667.png)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071521323.png)
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071526881.png)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071530795.png)

![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071530554.png)



讲讲从setContentView到activity显示布局的流程

## 5.1 概述

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324163158)

## 5.2 问题

### 5.2.1 为什么调用setContentView就能将布局显示出来？

调用setContentView方法内部会调用PhoneWindow的setContentView方法，其内部通过

`mLayoutInflater.inflate(layoutResID, mContentParent);`加载到DecorView的子布局mContentParent中，而DecorView是我们的顶级View，会在Activity启动后加载到当前Activity的应用程序窗口，所以我们调用setContentView就能将我们的布局显示出来



### 5.2.2 为什么requestFeature需要在setContentView之前调用？

当我们在Activity中调用了setContentView方法，会调用PhoneWindow的generateLayout方法，该方法会根据requestFeature方法设置的属性来选择DecorView中加载的布局，以及根据一些特性，例如是否显示标题，来设置当前窗口的特性。



### 5.2.3 PhoneWindow和Window之间有什么关系？

当我们在Activity中调用了setContentView方法，内部会调用Window的setContentView方法，Window是一个抽象类，而PhoneWindow是抽象类Window的唯一子类。Window的实例必须当做顶级View添加到WindowManager中。



### 5.2.4 DecorView和我们的布局有什么关系？

DecorView是我们窗口的顶级View，意味着我们使用Hierarchy Viewer查看View的层级关系时，最上层的View都是DecorView。我们的布局是加载在DecorView下的一个id为content的FrameLayout中的。



### 5.2.5 为什么继承AppCompactActivity后，主题需要继承AppCompactTheme？

在AppCompactActivity中调用setContentView，内部会调用AppCompatDelegateImplV9的createSubDecor方法，其中会加载兼容Window的主题`AppCompactTheme`。




# 6 问题

## 6.1 面试题
![](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202303071517835.png)

## 6.2 View的绘制流程是从Activity的哪个生命周期方法开始执行的





## 6.3 在Activity中获取某个View的宽高(如何在onCreate中拿到View的宽度和高度)
由于View的measure过程和Activity的生命周期方法不是同步执行的，如果View还没有测量完毕，那么获得的宽/高就是0。所以在onCreate、onStart、onResume中均无法正确得到某个View的宽高信息。解决方式如下：

- Activity/View#onWindowFocusChanged

```
// 此时View已经初始化完毕
// 当Activity的窗口得到焦点和失去焦点时均会被调用一次
// 如果频繁地进行onResume和onPause，那么onWindowFocusChanged也会被频繁地调用
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    if (hasFocus) {
        int width = view.getMeasureWidth();
        int height = view.getMeasuredHeight();
    }
}
```

- view.post(runnable)

利用Handler通信机制，发送一个Runnable到MessageQueue中，当View布局处理完成时，自动发送消息，通知UI进程。借此机制，巧妙获取View的高宽属性，代码简洁，相比ViewTreeObserver监听处理，还不需要手动移除观察者监听事件。

```
// 通过post可以将一个runnable投递到消息队列的尾部，// 然后等待Looper调用次runnable的时候，View也已经初
// 始化好了
protected void onStart() {
    super.onStart();
    view.post(new Runnable() {

        @Override
        public void run() {
            int width = view.getMeasuredWidth();
            int height = view.getMeasuredHeight();
        }
    });
}
```

- ViewTreeObserver
监听View的onLayout()绘制过程，一旦layout触发变化，立即回调onLayoutChange方法。 注意，使用完也要主要调用removeOnGlobalListener()方法移除监听事件。避免后续每一次发生全局View变化均触发该事件，影响性能。

```
// 当View树的状态发生改变或者View树内部的View的可见// 性发生改变时，onGlobalLayout方法将被回调
protected void onStart() {
    super.onStart();

    ViewTreeObserver observer = view.getViewTreeObserver();
    observer.addOnGlobalLayoutListener(new OnGlobalLayoutListener() {

        @SuppressWarnings("deprecation")
        @Override
        public void onGlobalLayout() {
            view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
            int width = view.getMeasuredWidth();
            int height = view.getMeasuredHeight();
        }
    });
}
```

- View.measure(int widthMeasureSpec, int heightMeasureSpec)

## 6.4 View执行onMeasure,onLayout的次数

分析ViewRootImpl的源码，scheduleTraversales()内部会执行postCallBack触发mTraversalRunnable重新走一遍performTraversals(),第二次执行performTraversals()就会触发performDraw()。所以performTraversals()走了两次，那么肯定会走2次measure方法，但不一定走2次onMeasure()，读过View measure方法源码的都应知道measure方法做了2级测量优化：

- 1.如果flag不为forceLayout或者与上次测量规格（MeasureSpec）相比未改变，那么将不会进行重新测量（执行onMeasure方法），直接使用上次的测量值；
- 2.如果满足非强制测量的条件，即前后二次测量规格不一致，会先根据目前测量规格生成的key索引缓存数据，索引到就无需进行重新测量;如果targetSDK小于API 20则二级测量优化无效，依旧会重新测量，不会采用缓存测量值。



## 6.5 getWidth()和getMeasuredWidth()的区别
getMeasuredWidth()、getMeasuredHeight()必须在onMeasure之后使用才有效）getMeasuredWidth() 的取值最终来源于 **setMeasuredDimension**() 方法调用时传递的参数, getWidth()返回的是，mRight - mLeft，mRight、mLeft 变量分别表示 View 相对父容器的左右边缘位置，getWidth()与getHeight()方法必须在**layout**(int l, int t, int r, int b)执行之后才有效


## 6.6 自定义View执行invalidate()方法,为什么有时候不会回调onDraw()



## 6.7 invalidate和postInvalidate区别

二者都会出发刷新View，并且当这个View的可见性为VISIBLE的时候，View的onDraw()方法将会被调用，invalidate()方法在 UI 线程中调用，重绘当前 UI。postInvalidate() 方法在非 UI 线程中调用，通过Handler通知 UI 线程重绘。



## 6.8 Requestlayout，onlayout，onDraw，DrawChild区别与联系



## 6.9 requestLayout()的作用

requestLayout()也可以达到重绘view的目的，但是与前两者不同，它会先调用onLayout()重新排版，再调用ondraw()方法。当view确定自身已经不再适合现有的区域时，该view本身调用这个方法要求parent view（父类的视图）重新调用他的onMeasure、onLayout来重新设置自己位置。特别是当view的layoutparameter发生改变，并且它的值还没能应用到view上时，这时候适合调用这个方法requestLayout()。 



## 6.10 onDraw() 和dispatchDraw()的区别
- 绘制View本身的内容，通过调用View.onDraw(canvas)函数实现
- 绘制自己的孩子通过dispatchDraw（canvas）实现

draw过程会调用onDraw(Canvas canvas)方法，然后就是dispatchDraw(Canvas canvas)方法, dispatchDraw()主要是分发给子组件进行绘制，我们通常定制组件的时候重写的是onDraw()方法。值得注意的是ViewGroup容器组件的绘制，当它没有背景时直接调用的是dispatchDraw()方法, 而绕过了draw()方法，当它有背景的时候就调用draw()方法，而draw()方法里包含了dispatchDraw()方法的调用。因此要在ViewGroup上绘制东西的时候往往重写的是dispatchDraw()方法而不是onDraw()方法，或者自定制一个Drawable，重写它的draw(Canvas c)和 getIntrinsicWidth()方法，然后设为背景。



## 6.11 requestlayout,onlayout,onDraw,DrawChild区别与联系
requestLayout()方法 ：会导致调用measure()过程 和 layout()过程 。 将会根据标志位判断是否需要ondraw

onLayout()方法(如果该View是ViewGroup对象，需要实现该方法，对每个子视图进行布局)

调用onDraw()方法绘制视图本身 (每个View都需要重载该方法，ViewGroup不需要实现该方法)

drawChild()去重新回调每个子视图的draw()方法



## 6.12 requestLayout，invalidate 的区别





## 6.13 如何更新UI?
android中有下列几种异步更新ui的解决办法：

* Activity.runOnUiThread(Runnable)
* View.post(Runnable)
* View.postDelayed(Runnable, long)
* 使用handler（线程间通讯）（推荐）
* AsyncTask（推荐）



Activity.runOnUiThread

```
public void onClick(View v) { 
    new Thread(new Runnable() { 
        public void run() { 
           Bitmap bitmap = loadImageFromNetwork("http://example.com/image.png");
           runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    mImageView.setImageBitmap(bitmap);  
                }
           });              
        } 
    }).start(); 
}
```



View.post(Runable)

```
public void onClick(View v) { 
    new Thread(new Runnable() { 
        public void run() { 
           Bitmap bitmap = loadImageFromNetwork("http://example.com/image.png");
           imageView.post(new Runnable() {
              @Override
              public void run() {
                  mImageView.setImageBitmap(bitmap);  
              }
           });             
        } 
    }).start(); 
}
```



 View.postDelayed(Runnable, long)

```
public void onClick(View v) { 
    new Thread(new Runnable() { 
        public void run() { 
           Bitmap bitmap = loadImageFromNetwork("http://example.com/image.png");  
           imageView.postDelayed(new Runnable() {
              @Override
              public void run() {
                  mImageView.setImageBitmap(bitmap);  
              }
           },2000);          
        } 
    }).start(); 
}
```

使用Handler（推荐）

```
new Thread(new Runnable() { 
    public void run() { 
        Bitmap bitmap = loadImageFromNetwork("http://example.com/image.png");  
        Message message = mHandler.obtainMessage();
        message.what = 1;
        message.obj = bitmap;
        mHandler.sendMessage(message);        
    } 
}).start();  


Handler mHandler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what){
            case 1:
                Bitmap bitmap = (Bitmap) msg.obj;
                imageView.setImageBitmap(bitmap);
                break;
            case 2:
                // ...
                break;
            default:
                break;
        }
    }
};
```



AsyncTask（推荐）

```
AsyncTask<String,Void,Bitmap> asyncTask = new AsyncTask<String, Void, Bitmap>() {

    /**
     * 即将要执行耗时任务时回调，这里可以做一些初始化操作
     */
    @Override
    protected void onPreExecute() {
        super.onPreExecute();
    }

    /**
     * 在后台执行耗时操作，其返回值将作为onPostExecute方法的参数
     * @param params
     * @return
     */
    @Override
    protected Bitmap doInBackground(String... params) {
        Bitmap bitmap = loadImageFromNetwork(params[0]);
        return bitmap;
    }

    /**
     * 当这个异步任务执行完成后，也就是doInBackground（）方法完成后，
     * 其方法的返回结果就是这里的参数
     * @param bitmap
     */
    @Override
    protected void onPostExecute(Bitmap bitmap) {
        imageView.setImageBitmap(bitmap);
    }
};
asyncTask.execute("http://example.com/image.png");
```



## 6.14 为什么子线程不能更新UI？

- 众所周知在`Android`中，子线程是不能更新`UI`的；

- 那么我在想，为什么不能，会产生什么问题；

- 是否真的就一定不能在子线程更新`UI`

  

### 6.14.1 什么是UI线程

Android的核心进程zygote进程fork出我们的app，app启动的最终会走入到ActivityThread中的main方法，在main方法中会调用Looper。其中ActivityThread所在的线程被称为UI线程，也就是我们常说的主线程 (Main thread)。 关于Main thread这个称呼其实可以查看ActivityThread中main方法的源码：

```
 public static void main(String[] args) {
        ... ...
        //注释1
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
复制代码
```

注释1：只看最后两行代码，在Loop.loop();调用完直接抛出异常,这里异常提到了当前线程称之为Main thread.

### 6.14.2 UI线程的工作机制

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210324161345)



### 6.14.3 能否在子线程中更新UI

答案是可以的，比如以下代码：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        tv = findViewById(R.id.tv);
        new Thread(new Runnable() {
            @Override
            public void run() {
                tv.setText("测试是否报出异常");
            }
        }).start();
    }
```

运行结果并无异常，可以正常的在子线程中更新了`TextView`控件;假如让线程休眠`1000ms`,就会发生错误：

> Only the original thread that created a view hierarchy can touch its views.



在该方法中，终于看到了`ViewRootImpl`的创建；

> **结论:**从以上的源码分析可得知，`ViewRootImpl`对象是在`onResume`方法回调之后才创建，那么就说明了为什么在生命周期的`onCreate`方法里，甚至是`onResume`方法里都可以实现子线程更新UI，因为此时还没有创建`ViewRootImpl`对象，并不会进行是否为主线程的判断；



### 6.14.4 更新UI一定要在主线程实现

谷歌提出：“一定要在主线程更新`UI`”，实际是为了提高界面的效率和安全性，带来更好的流畅性；反推一下，假如允许多线程更新`UI`，但是访问`UI`是没有加锁的，一旦多线程抢占了资源，那么界面将会乱套更新了，体验效果就不言而喻了；所以在`Android`中规定必须在主线程更新`UI`。



### 6.14.5 总结

- 子线程可以在`ViewRootImpl`还没有被创建之前更新`UI`；
- 访问`UI`是没有加对象锁的，在子线程环境下更新`UI`，会造成不可预期的风险；
- 开发者更新`UI`一定要在主线程进行操作;



### 6.14.6 为什么只能有一个线程操作 UI？
两个线程不能同时draw，否则屏幕会花；
不能同时insert map，否则内存会花；
不能同时write buffer，否则文件会花。
需要互斥，比如锁。多线程操作一个UI，很容易导致，或者极其容易导致反向加锁和死锁问题。

结果就是同一时刻只有一个线程可以做ui。那么当两个线程互斥几率较大时，或者保证互斥的代码复杂时，选择其中一个长期持有其他发消息就是典型的解决方案。所以普遍的要求ui只能单线程

源码分析：

如果在子线程更新 UI：

```java
new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                main_tv.setText("子线程中访问");
            }
        }).start();
```

Crash msg：

```
android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:6581)
at android.view.ViewRootImpl.requestLayout(ViewRootImpl.java:924)
```

`ViewRootImpl` 的 `checkThread`方法:

```java
void checkThread() {
    // mThread是主线程，在应用程序启动的时候，就已经被初始化了
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

`requestLayout` 方法:

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

`scheduleTraversals()`:

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

注意到postCallback方法的的第二个参数传入了很像是一个后台任务。那再点进去

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

可以看到里面调用了一个 performTraversals() 方法，View 的绘制过程就是从这个 performTraversals 方法开始的。分析到了这里，其实异常信息对我们帮助也不大了，它只告诉了我们子线程中访问UI在哪里抛出异常。

当访问UI时，ViewRootImpl 会调用 checkThread 方法去检查当前访问UI的线程是哪个，如果不是UI线程则会抛出异常，这是没问题的。但是为什么一开始在 MainActivity 的 onCreate方法中创建一个子线程访问UI，程序还是正常能跑起来呢？

唯一的解释就是执行 onCreate 方法的那个时候 ViewRootImpl 还没创建，无法去检查当前线程。

那么就可以这样深入进去。寻找 ViewRootImpl 是在哪里，是什么时候创建的



#### 6.14.6.1 大致流程

* 第一步，查看：ActivityThread --> handleResumeActivity
* handleResumeActivity 调用了 performResumeActivity
* performResumeActivity 调用了 r.activity.performResume()
* Instrumentation调用了callActivityOnResume方法
* activity调用了makeVisible
* 往WindowManager中添加DecorView
* WindowManagerImpl的addView
* WindowManagerGlobal的addView方法
* ViewRootImpl --> root.setView(view, wparams, panelParentView);





## 6.15 测量`View`的宽 / 高

1. 在某些情况下，需要多次测量`（measure）`才能确定`View`最终的宽/高；
2. 该情况下，`measure`过程后得到的宽 / 高可能不准确；
3. 此处建议：在`layout`过程中`onLayout()`去获取最终的宽 / 高



### 6.15.1 在onResume中可以测量宽高么？





## 6.16 View.inflater过程与异步inflater



## 6.17 inflater为什么比自定义View慢



## 6.18 LinearLayout和RelativeLayout性能对比

1. RelativeLayout会让子View调用2次onMeasure，LinearLayout 在有weight时，也会调用子View2次onMeasure
2. RelativeLayout的子View如果高度和RelativeLayout不同，则会引发效率问题，当子View很复杂时，这个问题会更加严重。如果可以，尽量使用padding代替margin。
3. 在不影响层级深度的情况下,使用LinearLayout和FrameLayout而不是RelativeLayout。

最后再思考一下文章开头那个矛盾的问题，为什么Google给开发者默认新建了个RelativeLayout，而自己却在DecorView中用了个LinearLayout。因为DecorView的层级深度是已知而且固定的，上面一个标题栏，下面一个内容栏。采用RelativeLayout并不会降低层级深度，所以此时在根节点上用LinearLayout是效率最高的。而之所以给开发者默认新建了个RelativeLayout是希望开发者能采用尽量少的View层级来表达布局以实现性能最优，因为复杂的View嵌套对性能的影响会更大一些。



## 6.19 事件冲突怎么解决？





# 7 view.post() 的四大常见疑问

* 为什么view.post()能保证获取到view的宽高？

* 为什么onCreate()使用view.post()无法立刻执行任务（如获取宽高）

* 若只是创建一个 View & 调用view.post()传入要执行的任务，为什么该任务不会被执行？

* view.pos()传入的任务被执行的有效期是什么时间节点？

* 

  常见疑问1

  a. 描述
  为什么view.post()能保证获取到view的宽高？



## 7.1 常见疑问1

### 7.1.1 a. 描述
为什么view.post()能保证获取到view的宽高？



### 7.1.2 b. 原因

View.post()的原理：以Handler为基础，View.post() 将传入任务添加到 View绘制任务所在的消息队列尾部，从而保证View.post() 任务的执行时机是在View 绘制任务完成之后的。 其中，几个关键点：

View.post()实际操作：将view.post()传入的任务保存到一个数组里

View.post()添加的任务 添加到 View绘制任务所在的消息队列尾部的时机：View 绘制流程的开始阶段，即 ViewRootImpl.performTraversals()

View.post()添加的任务执行时机：在View绘制任务之后

所以：

通过View.post()添加的任务是在View绘制任务里 - 开始绘制阶段时添加到消息队列尾部的；

所以，View.post() 添加的任务的执行是在View绘制任务后才执行，即在View绘制流程结束之后执行。

即View.post() 添加的任务能够保证在所有 View绘制流程结束之后才被执行，所以 执行View.post() 添加的任务时可以正确获取到 View 的宽高。

具体源码分析请看：Android：为什么view.post()能保证获取到view的宽高？(https://carsonho.blog.csdn.net/article/details/109282606)



## 7.2 常见疑问2

### 7.2.1 a. 描述
为什么onCreate()使用view.post()无法立刻执行任务（如获取宽高），需要在onResume()后才可获取？

### 7.2.2 b 原因
在onCreate()时，AttachInfo还没被赋值（为null）（是在view.dispatchAttachedToWindow()才被赋值），所以会走下述源码的过程2；通过上面分析，此过程的作用仅是：保存了通过post()添加的任务，并没执行。

public boolean post(Runnable action) {
    
    // ...
    
    // 判断AttachInfo是否为null
    final AttachInfo attachInfo = mAttachInfo;
    
    // 过程1：若不为null,直接调用其内部Handler的post ->>分析1
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }
    
    // 过程2：若为null，则加入当前View的等待队列
    getRunQueue().post(action); 
    return true;
}

### 7.2.3 c. 实例代码演示

    @Override
    public void onCreate(Bundle savedInstanceState) {
    // 执行日志1：carsonhe oncreate()
    
    view.post(new Runnable() {
            @Override
            public void run() {
    
                // 执行日志2：carsonhe view.post() do something
    
            }
        });
    }
    
    @Override
    protected void onResume() {
        // 执行日志3：carsonhe onresume()
    }
    
    // 输出日志展示
    日志1：carsonhe oncreate()
    日志3：carsonhe onresume()
    日志2：carsonhe view.post() do something

## 7.3 常见疑问3
### 7.3.1 a. 问题描述

若只是创建一个 View & 调用它的post()，那么post的任务会不会被执行？

    final View view = new View(this);
    
        view.post(new Runnable() {
            @Override
            public void run() {
                // ...
            }
        });

### 7.3.2 b. 答案

不会。主要原因是：
每个View中post() 需执行的任务，必须得添加到窗口视图-执行绘制流程 - 任务才会被post到消息队列里去等待执行，即依赖于dispatchAttachedToWindow ()；

若View未添加到窗口视图，那么就不会走绘制流程，post() 添加的任务最终不会被post到消息队列里，即得不到执行。（但会保存到HandlerAction数组里）

上述例子，因为它没有被添加到窗口视图，所以不会走绘制流程，所以该任务最终不会被post到消息队列里 & 执行

### 7.3.3 c. 解决方案

此时只需要添加将View添加到窗口，那么post()的任务即可被执行

// 因为此时会重新发起绘制流程，post的任务会被放到消息队列里，所以会被执行

```
contentView.addView(view);
```


## 7.4 常见疑问4

### 7.4.1 描述
view.pos()传入的任务被执行的有效期是多久？

### 7.4.2 结论
在整个 Activity 的生命周期内都可以正常使用 View.post() 任务

### 7.4.3 原因
任务被执行是构造AttachInfo，所以任务释放即时释放AttachInfo （置为null）。而AttachInfo 的释放操作（置为null）是在 Activity 生命周期 onDestory 方法之后

### 7.4.4 原因分析
目标
跟踪 AttachInfo 的释放过程（即何时置为null）

方向
AttachInfo的赋值依赖于DecorView.dispatchAttachedToWindow()，那么释放过程，容易联想到是对应的：DecorView.dispatchDetachedFromWindow()

具体源码分析

```java
  * 入口分析：DecorView.dispatchDetachedFromWindow()
  * 实际上是调用父类ViewGroup.dispatchDetachedFromWindow()
    

    /**
      * 入口分析：DecorView.dispatchDetachedFromWindow()
      * 实际上是调用父类ViewGroup.dispatchDetachedFromWindow()
        */
    
      void dispatchDetachedFromWindow() {
    
        // ... 
        
        final int count = mChildrenCount;
        final View[] children = mChildren;
        
        // 遍历所有childView
        for (int i = 0; i < count; i++) {
            // 遍历所有childView & dispatchDetachedFromWindow()
            // 分析1
            children[i].dispatchDetachedFromWindow();
        }
    }
    
    /**
      * 分析1：childView.dispatchDetachedFromWindow()
        */
        void dispatchDetachedFromWindow() {
    
        // ... 
    
        AttachInfo info = mAttachInfo;
       
        // 1. 回调View.onDetachedFromWindow()
        onDetachedFromWindow();
        
        // 2. 通知所有监听View.onAttachToWindow的监听者回调onViewDetachedFromWindow()
        ListenerInfo li = mListenerInfo;
        final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners = li != null ? li.mOnAttachStateChangeListeners : null;
        if (listeners != null && listeners.size() > 0) {
            for (OnAttachStateChangeListener listener : listeners) {
                listener.onViewDetachedFromWindow(this);
            }
        }
    
        // 3. 将AttachInfo置为null
        mAttachInfo = null;
        }
```

下面，我们将分析，什么时候调用上述入口，即DecorView.dispatchDetachedFromWindow()；

此时需从 将DecorView从WindowManager中移除 开始讲起：移除 Window 窗口任务是通过 ActivityThread.handleDestoryActivity()完成。

```java
/**
 * 入口
 */
private void handleDestroyActivity(IBinder token, boolean finishing,
        int configChanges, boolean getNonConfigInstance) {

    // 关注1：回调 Activity.onDestory()
    ActivityClientRecord r = performDestroyActivity(token, finishing,
            configChanges, getNonConfigInstance);

    // 获取当前Window的WindowManager
    WindowManager wm = r.activity.getWindowManager();
    // 当前Window的DecorView
    View v = r.activity.mDecor;
       
    // 关注2：通知WindowManager,移除当前 Window窗口
    wm.removeViewImmediate(v);
    // 此处即会释放AttachInfo
    // 因为在关注1处是在回调 Activity.onDestory()后，故在整个Activity的生命周期内都可以正常使用 View.post() 任务
    // 下面继续分析如何移除 ->> 分析1
                
}

/**
 * 分析1：WindowManager.removeViewImmediate()
 */
public void removeViewImmediate(View view) {
    
    mGlobal.removeView(view, true);
    // 调用WindowManagerGlobal的removeView()
    // ->> 分析2
}

/**
 * 分析2：WindowManagerGlobal.removeView()
 */
public void removeView(View view, boolean immediate) {
    // ...
   
    // 找到保存该DecorView的下标
    int index = findViewLocked(view, true);

    // 找到对应的ViewRootImpl，内部的DecorView
    View curView = mRoots.get(index).getView();

    // 从WindowManager中移除该DecorView
    // immediate 表示是否立即移除
    removeViewLocked(index, immediate);
    // ->> 分析3

}

/**
 * 分析3
 */
private void removeViewLocked(int index, boolean immediate) {

    // 找到对应的ViewRootImpl
    ViewRootImpl root = mRoots.get(index);

    // 该View是DecorView
    View view = root.getView();

    // ... 

    // 调用ViewRootImpl的die
    // 并且将当前ViewRootImpl在WindowManagerGlobal中移除
    boolean deferred = root.die(immediate);
    // ->> 分析4
}

/**
 * 分析4
 */
boolean die(boolean immediate) {

    // immediate 表示立即执行
    // mIsInTraversal 表示是否正在执行绘制任务
    if (immediate && !mIsInTraversal) {
        
        doDie();
        // ->> 分析5
    }

    // ...
}

/**
  * 分析5
  */
void doDie() {

    // ...
    if (mAdded) {
        
        dispatchDetachedFromWindow();
        // 回调View的dispatchDetachedFromWindow
        // ->> 即一开始分析的DecorView.dispatchAttachedToWindow()
    }

    // 将其从WindowManagerGlobal中移除DecorView
    WindowManagerGlobal.getInstance().doRemoveView(this);
}
```

### 7.4.5 最终原因 & 结论

View.post() 任务被执行的有效期是在 Activity 生命周期 onDestory()后。本质是追踪AttachInfo的释放过程（置为null）

AttachInfo的释放过程是在 将DecorView从WindowManager中移除时：回调DecorView.dispatchDetachedFromWindow()，其具体行为是：

1、回调View.onDetachedFromWindow()
2、通知所有监听View.onAttachToWindow的监听者回调onViewDetachedFromWindow()
3、将AttachInfo置为null
而上述过程是在ActivityThread.handleDestoryActivity()中回调 Activity.onDestory()之后。


# 8 面试题

## 8.1 View的绘制流程,每个方法干什么的,如果要获取View的宽高,在哪个方法里获取


## 8.2 MeasureSpec是什么
MeasureSpec表示的是一个32位的整形值，它的高2位表示测量模式SpecMode，低30位表示某种测量模式下的 规格大小SpecSize。MeasureSpec是View类的一个静态内部类，用来说明应该如何测量这个View。它由三种测量模 式，如下:

**EXACTLY**:精确测量模式，视图宽高指定为match_parent或具体数值时生效，表示父视图已经决定了子视图的 精确大小，这种模式下View的测量值就是SpecSize的值。

**AT_MOST**:最大值测量模式，当视图的宽高指定为wrap_content时生效，此时子视图的尺寸可以是不超过父视 图允许的最大尺寸的任何尺寸。

**UNSPECIFIED**:不指定测量模式, 父视图没有限制子视图的大小，子视图可以是想要的任何尺寸，通常用于系统 内部，应用开发中很少用到。

MeasureSpec通过将SpecMode和SpecSize打包成一个int值来避免过多的对象内存分配，为了方便操作，其提 供了打包和解包的方法，打包方法为makeMeasureSpec，解包方法为getMode和getSize。


## 8.3 子View创建MeasureSpec创建规则是什么
根据父容器的MeasureSpec和子View的LayoutParams等信息计算子View的MeasureSpec

![image.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202304171130991.png)


## 8.4 自定义View的wrap_content不起作用的原因
1.因为onMeasure()->getDefaultSize()，当View的测量模式是AT_MOST或EXACTLY时，View的大小都会被设置 成子View MeasureSpec的specSize。
```java
public static int getDefaultSize(int size, int measureSpec) {
     switch (specMode) {

         case MeasureSpec.UNSPECIFIED:
            result = size;
            break;

         case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:

            result = specSize;
            break;

}

    return result;
}
```

2.View的MeasureSpec值是根据子View的布局参数(LayoutParams)和父容器的MeasureSpec值计算得来， 具体计算逻辑封装在getChildMeasureSpec()。 当子View wrap_content或match_parent情况下，子View MeasureSpec的specSize被设置成parenSize = 父容器当前剩余空间大小
![image.png](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/202304171131318.png)

3.所以当给一个View/ViewGroup设置宽高为具体数值或者match_parent，它都能正确的显示，但是如果你设置 的是wrap_content->AT_MOST，则默认显示出来是其父容器的大小。如果你想要它正常的显示为wrap_content，所 以需要自己重写onMeasure()来自己计算它的宽高度并设置。此时，可以在wrap_content的情况下(对应 MeasureSpec.AT_MOST)指定内部宽/高(mWidth和mHeight)。

## 8.5 在Activity中获取某个View的宽高有几种方法
- Activity/View#onWindowFocusChanged:此时View已经初始化完毕，当Activity的窗口得到焦点和失去焦 点时均会被调用一次，如果频繁地进行onResume和onPause，那么onWindowFocusChanged也会被频繁 地调用。  
- view.post(runnable): 通过post将runnable放入ViewRootImpl的RunQueue中，RunQueue中runnable 最后的执行时机，是在下一个performTraversals到来的时候，也就是view完成layout之后的第一时间获取 宽高。 
- ViewTreeObserver#addOnGlobalLayoutListener:当View树的状态发生改变或者View树内部的View的可 见性发生改变时，onGlobalLayout方法将被回调。
- View.measure(int widthMeasureSpec, int heightMeasureSpec): match_parent 直接放弃，无法 measure出具体的宽/高。原因很简单，根据view的measure过程，构造此种MeasureSpec需要知道 parentSize，即父容器的剩余空间，而这个时候我们无法知道parentSize的大小，所以理论上不可能测量处 view的大小。


wrap_content 
```java
int widthMeasureSpec = View.MeasureSpec.makeMeasureSpec((1<<30)-1, View.MeasureSpec.AT_MOST); int heightMeasureSpec = View.MeasureSpec.makeMeasureSpec((1<<30)-1, View.MeasureSpec.AT_MOST); v_view1.measure(widthMeasureSpec, heightMeasureSpec);
```

注意到(1<<30)-1，我们知道MeasureSpec的前2位为mode，后面30位为size，所以说我们使用最大size值去匹 配该最大化模式，让view自己去计算需要的大小。 这个特殊的 int 值就是 View 理论上能支持的最大值。 View 的尺 寸使用 30 位二进制来表示，也就是说最大是 30 个 1(即 2^30 -1)，也就是 (1<<30)-1。

具体的数值(dp/px) 这种模式下，只需要使用具体数值去measure即可，比如宽/高都是100px: 
```java
int widthMeasureSpec = View.MeasureSpec.makeMeasureSpec(100, View.MeasureSpec.EXACTLY); int heightMeasureSpec = View.MeasureSpec.makeMeasureSpec(100, View.MeasureSpec.EXACTLY); v_view1.measure(widthMeasureSpec, heightMeasureSpec);
```