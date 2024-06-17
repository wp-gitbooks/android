---
number headings: auto, first-level 1, max 6, 1.1
---

# 1 概念
https://xiaojianchen.gitbook.io/android-foundations/shi-jian-fen-fa#viewgroup-de-chuan-di-ji-zhi

事件类型：事件类型分为 ACTION_DOWN, ACTION_UP, ACTION_MOVE, ACTION_POINTER_DOWN, ACTION_POINTER_UP, ACTION_CANCEL 等
事件传递：事件分发、拦截、消费。Activity、View、ViewGroup



# 2 概述
## 2.1 什么是事件分发
事件分发是将屏幕触控信息分发给控件树的一个套机制。  
当我们触摸屏幕时，会产生一些列的MotionEvent事件对象，经过控件树的管理者ViewRootImpl，调用view的dispatchPointerEvnet方法进行分发。


## 2.2 分发流程 [[0-View概述]]
在程序的主界面情况下，布局的顶层view是DecorView，他会先把事件交给Activity，Activity调用PhoneWindow的方法进行分发，PhoneWindow会调用DecorView的父类ViewGroup的dispatchTouchEvent方法进行分发。也就是**Activity->Window->ViewGroup**的流程。ViewGroup则会向下去寻找合适的控件并把事件分发给他



## 2.3 事件一定会经过Activity吗？
不是的。我们的程序界面的顶层viewGroup，也就是decorView中注册了Activity这个callBack，所以当程序的主界面接收到事件之后会先交给Activity。  
但是，如果是另外的控件树，如dialog、popupWindow等事件流是不会经过Activity的。只有自己界面的事件才会经Activity。



## 2.4 Activity的分发方法中调用了onUserInteraction()方法，你能说说这个方法有什么作用吗？
好的。这个方法在Activity接收到down的时候会被调用，本身是个空方法，需要开发者自己去重写。  
通过官方的注释可以知道，这个方法会在我们以任意的方式**开始**与Activity进行交互的时候被调用。比较常见的场景就是屏保：当我们一段时间没有操作会显示一张图片，当我们开始与Activity交互的时候可在这个方法中取消屏保；另外还有没有操作自动隐藏工具栏，可以在这个方法中让工具栏重新显示。



## 2.5 前面你讲到最后会分发到viewGroup，那么viewGroup是如何分发事件的？
viewGroup处理事件信息分为三个步骤：拦截、寻找子控件、派发事件。

事件分发中有一个重要的规则：一个触控点的一个事件序列只能给一个view处理，除非异常情况。所以如果viewGroup消费了down事件，那么子view将无法收到任何事件。

viewGroup第一步会判读这个事件是否需要分发给子view，如果是则调用onInterceptTouchEvent方法判断是否要进行拦截。  
第二步是如果这个事件是down事件，那么需要为他寻找一个消费此事件的子控件，如果找到则为他创建一个TouchTarget。  
第三步是派发事件，如果存在TouchTarget，说明找到了消费事件序列的子view，直接分发给他。如果没有则交给自己处理。



## 2.6 你前面讲到“一个触控点的一个事件序列只能给一个view处理，除非异常情况”,这里有什么异常情况呢？如果发生异常情况该如何处理？
这里的异常情况主要有两点：1.被viewGroup拦截，2.出现界面跳转等其他情况。

当事件流中断时，viewGroup会发送一个ACTION_CANCEL事件给到view，此时需要做一些状态的恢复工作，如终止动画，恢复view大小等等。

### 2.6.1 那既然说到ACTION_CANCEL类型，那你可以说说还有什么事件类型吗？
除了ACTION_CANCEL，其他事件类型还有：

-   ACTION_MOVE：当我们手指在屏幕上滑动时产生此事件
-   ACTION_UP：当我们手指抬起时产生此事件

此外多指操作也比较常见：

-   ACTION_POINTER_DOWN: 当已经有一个手指按下的情况下，另一个手指按下会产生该事件
-   ACTION_POINTER_UP: 多个手指同时按下的情况下，抬起其中一个手指会产生该事件。

一个完整的事件序列是从 ACTION_DOWN 开始，到 ACTION_UP 或者ACTION_CANCEL 结束。  
**一个手指**的完整序列是从 ACTION_DOWN/ACTION_POINTER_DOWN 开始，到ACTION_UP/ACTION_POINTER_UP/ACTION_CANCEL 结束。


## 2.7 说到多指，那你知道ViewGroup是如何将多个手指产生的事件准确分发给不同的子view吗
这个问题的关键在于 MotionEvent 以及 ViewGroup 内部的 TouchTarget。

每个MotionEvent中都包含了当前屏幕所有触控点的信息，他的内部用了一个数组来存储不同的触控id所对应的坐标数值。

当一个子view消费了down事件之后，ViewGroup会为该view创建一个TouchTarget，这个TouchTarget就包含了该view的实例与触控id。这里的触控id可以是多个，也就是一个view可接受多个触控点的事件序列。

当一个MotionEvent到来之时，ViewGroup会将其中的触控点信息拆开，再分别发送给感兴趣的子view。从而达到精准发送触控点信息的目的。


## 2.8 那 view 支持处理多指信息吗？

View默认是不支持的。他在获取触控点信息的时候并没有传入触控点索引，也就是获取的是MotionEvent内部数组中的第一个触控点的信息。多指需要我们自己去重写方法支持他。

## 2.9 那 View 是如何处理触摸事件的？
首先，他会判断是否存在onTouchListener，存在则会调用他的onTouch方法来处理事件。如果该方法返回true那么就分发结束直接返回。而如果该监听器为null或者onTouch方法返回了false，则会调用onTouchEvent方法来处理事件。

onTouchEvent方法中支持了两种监听器：onClickListener和onLongClickListener。View会根据不同的触摸情况来调用这两个监听器。同时进入到onTouchEvent方法中，无论该view是否是enable，只要是clickable，他的分发方法都是返回true。

总结一下就是：先调用onTouchListener，再调用onClickListener和onLongClickListener。

## 2.10 你前面多次讲到分发方法和返回值，那你可以讲讲主要有什么方法以及他们之间的关系吗？
嗯嗯。核心的方法有三个：dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent。

简单来说：dispatchTouchEvent是核心的分发方法，所有分发逻辑都在这个方法中执行；onInterceptTouchEvent在viewGroup负责判断是否拦截；onTouchEvent是消费事件的核心方法。viewGroup中拥有这三个方法，而view没有onInterceptTouchEvent方法。

-   viewGroup
    1.  viewGroup的dispatchTouchEvent方法接收到事件消息，首先会去调用onInterceptTouchEvent判断是否拦截事件
        -   如果拦截，则调用自身的onTouchEvent方法
        -   如果不拦截则调用子view的dispatchTouchEvent方法
    2.  子view没有消费事件，那么会调用viewGroup本身的onTouchEvent
    3.  上面1、2步的处理结果为viewGroup的dispatchTouchEvent方法的处理结果，没有消费则返回false并返回给上一层的onTouchEvent处理，如果消费则分发结束并返回true。
-   view
    1.  view的dispatchTouchEvent默认情况下会调用onTouchEvent来处理事件，返回true表示消费事件，返回false表示没有消费事件
    2.  第1步的结果就是dispatchTouchEvent方法的处理结果，成功消费则返回true，没有消费则返回false并交给上一层的onTouchEvent处理

简单来说，在控件树中，每个viewGroup在dispatchTouchEvent方法中不断往下分发寻找消费的view，如果底层的view没有消费事件则会一层层网上调用viewGroup的onTouchEvent方法来处理事件。

同时，由于Activity继承了Window.CallBack接口，所以也有dispatchTouchEvent和onTouchEvent方法：

1.  activity接收到触摸事件之后，会直接把触摸事件分发给viewGroup
2.  如果viewGroup的dispatchTouchEvent方法返回false，那么会调用Activity的onTouchEvent来处理事件
3.  第1、2步的处理结果就是activity的dispatchTouchEvent方法的处理结果，并返回给上层



## 2.11 看来你对事件分发了解得挺多的，那你在实际中有运用到事件分发吗？

嗯嗯，有的。举两个例子。

第一个需求是要设计一个按钮块，按下的时候会缩小高度变低同时变得半透明，放开的时候又会回弹。这个时候就可以在这个按钮的onTouchEvent方法中判断事件类型：down则开启按下动画，up则开启释放动画。同时注意接收到cancel事件的时候要恢复状态。

第二个是滑动冲突。解决滑动冲突的核心思路就是把滑动事件根据具体的情况分发给viewGroup或者内部view。主要的方法有外部拦截法和内部拦截法。  
外部拦截法的思路就是在viewGroup中判断滑动的情况，对符合自身滑动的事件进行拦截，对不符合的事件不拦截，给到内部view。内部拦截法的思路要求viewGroup拦截除了down事件以外的所有事件，然后再内部view中判断滑动的情况，对符合自身滑动情况的时间设置禁止拦截标志，对不符合自身滑动情况的事件则取消标志让viewGroup进行拦截。


### 2.11.1 那外部和内部拦截法该如何选择呢？
在一般的情况下，外部拦截法不需要对子view进行方法重写，比内部拦截法更加简单，推荐使用外部拦截法。

但如果需要在子view判断更多的触摸情况时，则使用内部拦截法可更加方法子view处理情况。

## 2.12 前面一直聊到触摸事件，那你知道一个触摸事件是如何从触摸屏幕开始产生的吗？






# 3 事件分发机制（原理）

## 3.1 核心类

View,ViewGroup,Activity都能处理Touch事件, 它们之间处理的先后顺序和方法有所不同.

| 方法      | dispatchTouchEvent | onInterceptTouchEvent | onTouchEvent |
| :-------- | :----------------- | :-------------------- | :----------- |
| Activity  | √                  | ×                     | √            |
| ViewGroup | √                  | √                     | √            |
| View      | √                  | ×                     | √            |

以上3个方法功能如下：

- dispatchTouchEvent(MotionEvent event)， 事件分发
- onInterceptTouchEvent(MotionEvent ev)， 事件拦截，该方法只有ViewGroup所拥有
- onTouchEvent(MotionEvent event)， 事件响应

1）View是所有视图对象的父类，实现了动画相关的接口Drawable.Callback， 按键相关的接口KeyEvent.Callback， 交互相关的接口AccessibilityEventSource。比如Button继承自View。

2）ViewGroup，是一个抽象类，一组View的集合，可以包含View和ViewGroup,是所有布局的父类或间接父类。继承了View，实现了ViewParent（用于与父视图交互的接口）， ViewManager（用于添加、删除、更新子视图到Activity的接口）。比如常用的LinearLayout，RelativeLayout都是继承自ViewGroup。

3）Activity是Android四大基本组件之一，当手指触摸到屏幕时，屏幕硬件一行行不断地扫描每个像素点，获取到触摸事件后，从底层产生中断上报。再通过native层调用Java层InputEventReceiver中的dispatchInputEvent方法。经过层层调用，交由Activity的dispatchTouchEvent方法来处理。



## 3.2 事件分发流程

1、函数dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent

2、返回值：true、false		消费、不消费



![touch](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210305143804.jpg)

1. `onInterceptTouchEvent`返回值true表示事件拦截， `onTouch/onTouchEvent` 返回值true表示事件消费。
2. 触摸事件先交由`Activity.dispatchTouchEvent`。再一层层往下分发，当中间的ViewGroup都不拦截时，进入最底层的View后，开始由最底层的`OnTouchEvent`来处理，如果一直不消费，则最后返回到`Activity.OnTouchEvent`。
3. ViewGroup才有`onInterceptTouchEvent`拦截方法。在分发过程中，中间任何一层ViewGroup都可以直接拦截，则不再往下分发，而是交由发生拦截操作的ViewGroup的`OnTouchEvent`来处理。
4. 子View可调用`requestDisallowInterceptTouchEvent`方法，来设置`disallowIntercept=true`，从而阻止父ViewGroup的`onInterceptTouchEvent`拦截操作。
5. OnTouchEvent由下往上冒泡时，当中间任何一层的OnTouchEvent消费该事件，则不再往上传递，表示事件已处理。
6. 如果View没有消费ACTION_DOWN事件，则之后的ACTION_MOVE等事件都不会再接收。
7. 只要`View.onTouchEvent`是可点击或可长按，则消费该事件.
8. `onTouch`优先于`onTouchEvent`执行，上面流程图中省略，`onTouch`的位置在`onTouchEvent`前面。当`onTouch`返回true,则不执行`onTouchEvent`,否则会执行onTouchEvent。`onTouch`只有View设置了`OnTouchListener`，且是enable的才执行该方法



# 4 事件分发机制（源码）

![input_event_dispatcher](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210308152357.jpg)

## 4.1 DecorView.dispatchTouchEvent

[-> PhoneWindow.java ::DecorView]

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Callback cb = getCallback();
    // [见小节2.2]
    return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
            : super.dispatchTouchEvent(ev);
}
```

此处cb是指Window的内部接口Callback. 对于Activity实现了Window.Callback接口. 故接下来调用Activity类.


## 4.2 Activity.dispatchTouchEvent

[-> Activity.java]

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        //第一次按下操作时，用户希望能与设备进行交互，可通过实现该方法
        onUserInteraction();
    }

    //获取当前Activity的顶层窗口是PhoneWindow [见小节2.3]
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    //当没有任何view处理时，交由activity的onTouchEvent处理
    return onTouchEvent(ev); // [见小节2.2.1]
}
```

如果重写Activity的dispatchTouchEvent()方法，则会在分发事件前可处理触摸事件的相关逻辑. 另外此处getWindow()返回的是Activity的mWindow成员变量，该变量赋值过程是在Activity.attach()方法, 可知其类型为PhoneWindow.


## 4.3 Activity.onTouchEvent

[-> Activity.java]

```Java
public boolean onTouchEvent(MotionEvent event) {
    //当窗口需要关闭时，消费掉当前event
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }

    return false;
}
```



## 4.4 superDispatchTouchEvent

[-> PhoneWindow.java]

```Java
public boolean superDispatchTouchEvent(KeyEvent event) {
    return mDecor.superDispatcTouchEvent(event); // [见小节2.4]
}
```

PhoneWindow的最顶View是DecorView，再交由DecorView处理。而DecorView的父类的父类是ViewGroup,接着调用 ViewGroup.dispatchTouchEvent()方法。为了精简篇幅，有些中间函数调用不涉及关键逻辑，可能会直接跳过。

## 4.5 ViewGroup.dispatchTouchEvent

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    boolean handled = false;
    //根据隐私策略而来决定是否过滤本次触摸事件,
    if (onFilterTouchEventForSecurity(ev)) {  // [见小节2.4.1]
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 发生ACTION_DOWN事件, 则取消并清除之前所有的触摸targets
            cancelAndClearTouchTargets(ev);
            resetTouchState(); // 重置触摸状态
        }

        // 发生ACTION_DOWN事件或者已经发生过ACTION_DOWN;才进入此区域，主要功能是拦截器
        //只有发生过ACTION_DOWN事件，则mFirstTouchTarget != null;
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            //可通过调用requestDisallowInterceptTouchEvent，不让父View拦截事件
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            //判断是否允许调用拦截器
            if (!disallowIntercept) {
                //调用拦截方法
                intercepted = onInterceptTouchEvent(ev);  // [见小节2.4.2]
                ev.setAction(action);
            } else {
                intercepted = false;
            }
        } else {
            // 当没有触摸targets，且不是down事件时，开始持续拦截触摸。
            intercepted = true;
        }
        ...

        //不取消事件，同时不拦截事件, 并且是Down事件才进入该区域
        if (!canceled && !intercepted) {
            //把事件分发给所有的子视图，寻找可以获取焦点的视图。
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;

            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // down事件等于0
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                removePointersFromTouchTargets(idBitsToAssign); //清空早先的触摸对象

                final int childrenCount = mChildrenCount;
                //第一次down事件，同时子视图不会空时
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // 从视图最上层到下层，获取所有能接收该事件的子视图
                    final ArrayList<View> preorderedList = buildOrderedChildList(); // [见小节2.4.3]
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;

                    /* 从最底层的父视图开始遍历， ** 找寻newTouchTarget，并赋予view与 pointerIdBits； ** 如果已经存在找寻newTouchTarget，说明正在接收触摸事件，则跳出循环。 */
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = customOrder
                                ? getChildDrawingOrder(childrenCount, i) : i;
                        final View child = (preorderedList == null)
                                ? children[childIndex] : preorderedList.get(childIndex);

                        // 如果当前视图无法获取用户焦点，则跳过本次循环
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }
                        //如果view不可见，或者触摸的坐标点不在view的范围内，则跳过本次循环
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        // 已经开始接收触摸事件,并退出整个循环。
                        if (newTouchTarget != null) {
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        //重置取消或抬起标志位
                        //如果触摸位置在child的区域内，则把事件分发给子View或ViewGroup
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) { // [见小节2.4.4]
                            // 获取TouchDown的时间点
                            mLastTouchDownTime = ev.getDownTime();
                            // 获取TouchDown的Index
                            if (preorderedList != null) {
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }

                            //获取TouchDown的x,y坐标
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            //添加TouchTarget,则mFirstTouchTarget != null。 [见小节2.4.5]
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            //表示以及分发给NewTouchTarget
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                        ev.setTargetAccessibilityFocus(false);
                    }
                    // 清除视图列表
                    if (preorderedList != null) preorderedList.clear();
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    //将mFirstTouchTarget的链表最后的touchTarget赋给newTouchTarget
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        // mFirstTouchTarget赋值是在通过addTouchTarget方法获取的；
        // 只有处理ACTION_DOWN事件，才会进入addTouchTarget方法。
        // 这也正是当View没有消费ACTION_DOWN事件，则不会接收其他MOVE,UP等事件的原因
        if (mFirstTouchTarget == null) {
            //没有触摸target,则由当前ViewGroup来处理
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            //如果View消费ACTION_DOWN事件，那么MOVE,UP等事件相继开始执行
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        //当发生抬起或取消事件，更新触摸targets
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    } //此处大括号，是if (onFilterTouchEventForSecurity(ev))的结尾

    //通知verifier由于当前时间未处理，那么该事件其余的都将被忽略
    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```



## 4.6 View.dispatchTouchEvent

[-> View.java]

```Java
public boolean dispatchTouchEvent(MotionEvent event) {
    ...

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        //在Down事件之前，如果存在滚动操作则停止。不存在则不进行操作
        stopNestedScroll();
    }

    // mOnTouchListener.onTouch优先于onTouchEvent。
    if (onFilterTouchEventForSecurity(event)) {
        //当存在OnTouchListener，且视图状态为ENABLED时，调用onTouch()方法
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true; //如果已经消费事件，则返回True
        }
        //如果OnTouch（)没有消费Touch事件则调用OnTouchEvent()
        if (!result && onTouchEvent(event)) { // [见小节2.5.1]
            result = true; //如果已经消费事件，则返回True
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    // 处理取消或抬起操作
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

1. 先由OnTouchListener的OnTouch()来处理事件，当返回True，则消费该事件，否则进入2。
2. onTouchEvent处理事件，的那个返回True时，消费该事件。否则不会处理



## 4.7 View.onTouchEvent

```
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;

    // 当View状态为DISABLED，如果可点击或可长按，则返回True，即消费事件
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        return (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
    }

    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    //当View状态为ENABLED，如果可点击或可长按，则返回True，即消费事件;
    //与前面的的结合，可得出结论:只要view是可点击或可长按，则消费该事件.
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        setPressed(true, x, y);
                   }

                    if (!mHasPerformedLongPress) {
                        //这是Tap操作，移除长按回调方法
                        removeLongPressCallback();

                        if (!focusTaken) {
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            //调用View.OnClickListener
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }

                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }

                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        mUnsetPressedState.run();
                    }

                    removeTapCallback();
                }
                break;

            case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                //获取是否处于可滚动的视图内
                boolean isInScrollingContainer = isInScrollingContainer();

                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    //当处于可滚动视图内，则延迟TAP_TIMEOUT，再反馈按压状态，用来判断用户是否想要滚动。默认延时为100ms
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    //当不再滚动视图内，则立刻反馈按压状态
                    setPressed(true, x, y);
                    checkForLongClick(0); //检测是否是长按
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                setPressed(false);
                removeTapCallback();
                removeLongPressCallback();
                break;

            case MotionEvent.ACTION_MOVE:
                drawableHotspotChanged(x, y);

                if (!pointInView(x, y, mTouchSlop)) {
                    removeTapCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        removeLongPressCallback();
                        setPressed(false);
                    }
                }
                break;
        }

        return true;
    }
    return false;
}
```

# 5 Android中onTouch，onTouchEvent，onClick优先级，关系

## 5.1 基础介绍

onTouch:指的是View设置的OnTouchListener接口的onTouch（）方法

onTouchEvent：指的是事件分发中的重要方法（dispatchTouchEvent，onInterceptTouchEvent，onTouchEvent）

onClick：指的是View设置的OnClickListener接口的onClick（）方法

## 5.2 源码分析
```
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    
    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    return result;
}
```

其中mOnTouchListener指的就是我们通过 setOnTouchListener（）设置的接口，可以知道如果onTouch（）返回true的话，事件就被消化，onTouchEvent（）就不会收到事件了，所以onTouch比onTouchEvent优先级高

接着看一下onTouchEvent（）

```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    
   
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
            
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }
                
                }
                mIgnoreNextUpEvent = false;
                break;

        }

        return true;
    }

    return false;
}
```

接着看看performClick（）

```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }

    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
```

从中可以看出在onTouchEvent（）中，调用了我们设置的OnClickListener()接口中的onClick（），所以onTouchEvent优先级高于onClick

## 5.3 总结

优先级从高到低：<font color='red'>onTouch>onTouchEvent>onClick</font>

onTouch() 方法的返回值决定了 onTouchEvent() 方法要不要执行，如果 onTouch() 返回 true，则 onTouchEvent() 不会再执行，返回 false ,则 onTouchEvent() 继续执行，而 onClick() 的回调是在 onTouchEvent() 方法中调用，onTouchEvent() 不执行则 onClick() 不执行。

![image-20210530141357286](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210530141357.png)


# 6 滑动冲突
https://www.jianshu.com/p/982a83271327


在开发当中，View 的滑动冲突时经常遇到的，比如 ViewPager 嵌套 ViewPager，ScrollView 嵌套 ViewPager。下面让我们一起来看看怎么解决。

## 6.1 常见的三种情况

第一种情况，滑动方向不同

![图片](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210528161342)

第二种情况，滑动方向相同

![图片](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210528161350)

第三种情况，上述两种情况的嵌套

![图片](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210528161407.png)

在这里插入图片描述







## 6.2 解决思路
看了上面三种情况，我们知道他们的共同特点是父View 和子View都想争着响应我们的触摸事件，但遗憾的是我们的触摸事件 同一时刻只能被某一个View或者ViewGroup拦截消费，所以就产生了滑动冲突。

那既然同一时刻只能由某一个 View 或者 ViewGroup 消费拦截，那我们就只需要 决定在某个时刻由这个 View 或者 ViewGroup 拦截事件，另外的 某个时刻由 另外一个 View 或者 ViewGroup 拦截事件，不就 OK了吗？

综上，正如 在 **《Android开发艺术》** 一书提出的，总共 有两种解决方案

以下解决思路来自于 **《Android开发艺术》** 书籍

**下面的两种方法针对第一种情况（滑动方向不同），父View是上下滑动，子View是左右滑动的情况。**

## 6.3 外部解决法

从父View着手，重写onInterceptTouchEvent方法，在父View需要拦截的时候拦截，不要的时候返回false，为代码大概 如下

```
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    final float x = ev.getX();
    final float y = ev.getY();

    final int action = ev.getAction();
    switch (action) {
        case MotionEvent.ACTION_DOWN:
            mDownPosX = x;
            mDownPosY = y;

            break;
        case MotionEvent.ACTION_MOVE:
            final float deltaX = Math.abs(x - mDownPosX);
            final float deltaY = Math.abs(y - mDownPosY);
            // 这里是够拦截的判断依据是左右滑动，读者可根据自己的逻辑进行是否拦截
            if (deltaX > deltaY) {
                return false;
            }
    }

    return super.onInterceptTouchEvent(ev);
}
```

## 6.4 内部解决法

从子View着手，父View先不要拦截任何事件，所有的事件传递给 子View，如果子View需要此事件就消费掉，不需要此事件的话就交给 父View处理。

实现思路 如下，重写子 View的dispatchTouchEvent方法，在Action_down 动作中通过方法 **requestDisallowInterceptTouchEvent（true） 先请求 父 View不要拦截事件**，这样保证子 View 能够接受到 Action_move 事件，再在 Action_move 动作中根据自己的逻辑是否要拦截事件，不需要拦截事件的话再交给 父 View 处理。

```
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    int x = (int) ev.getRawX();
    int y = (int) ev.getRawY();
    int dealtX = 0;
    int dealtY = 0;

    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            dealtX = 0;
            dealtY = 0;
            // 保证子View能够接收到Action_move事件
            getParent().requestDisallowInterceptTouchEvent(true);
            break;
        case MotionEvent.ACTION_MOVE:
            dealtX += Math.abs(x - lastX);
            dealtY += Math.abs(y - lastY);
            Log.i(TAG, "dealtX:=" + dealtX);
            Log.i(TAG, "dealtY:=" + dealtY);
            // 这里是够拦截的判断依据是左右滑动，读者可根据自己的逻辑进行是否拦截
            if (dealtX >= dealtY) {
                getParent().requestDisallowInterceptTouchEvent(true);
            } else {
                getParent().requestDisallowInterceptTouchEvent(false);
            }
            lastX = x;
            lastY = y;
            break;
        case MotionEvent.ACTION_CANCEL:
            break;
        case MotionEvent.ACTION_UP:
            break;

    }
    return super.dispatchTouchEvent(ev);
}
```

更多细节，可以查看我的这篇文章，里面有详细介绍哦
ViewPager，ScrollView 嵌套ViewPager滑动冲突解决



### 6.4.1 套路一 外部拦截法：

即父View根据需要对事件进行拦截。逻辑处理放在父View的onInterceptTouchEvent方法中。我们只需要重写父View的onInterceptTouchEvent方法，并根据逻辑需要做相应的拦截即可。

```java
    public boolean onInterceptTouchEvent(MotionEvent event) {
        boolean intercepted = false;
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                intercepted = false;
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                if (满足父容器的拦截要求) {
                    intercepted = true;
                } else {
                    intercepted = false;
                }
                break;
            }
            case MotionEvent.ACTION_UP: {
                intercepted = false;
                break;
            }
            default:
                break;
        }
        mLastXIntercept = x;
        mLastYIntercept = y;
        return intercepted;
    }
```

**上面伪代码表示外部拦截法的处理思路，需要注意下面几点**

> - 根据业务逻辑需要，在ACTION_MOVE方法中进行判断，如果需要父View处理则返回true，否则返回false，事件分发给子View去处理。

- ACTION_DOWN 一定返回false，不要拦截它，否则根据View事件分发机制，后续ACTION_MOVE 与 ACTION_UP事件都将默认交给父View去处理！
- 原则上ACTION_UP也需要返回false，如果返回true，并且滑动事件交给子View处理，那么子View将接收不到ACTION_UP事件，子View的onClick事件也无法触发。而父View不一样，如果父View在ACTION_MOVE中开始拦截事件，那么后续ACTION_UP也将默认交给父View处理！

### 6.4.2 套路二 内部拦截法：
即父View不拦截任何事件，所有事件都传递给子View，子View根据需要决定是自己消费事件还是给父View处理。这需要子View使用requestDisallowInterceptTouchEvent方法才能正常工作。下面是子View的dispatchTouchEvent方法的伪代码：


```java
    public boolean dispatchTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                parent.requestDisallowInterceptTouchEvent(true);
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                int deltaX = x - mLastX;
                int deltaY = y - mLastY;
                if (父容器需要此类点击事件) {
                    parent.requestDisallowInterceptTouchEvent(false);
                }
                break;
            }
            case MotionEvent.ACTION_UP: {
                break;
            }
            default:
                break;
        }

        mLastX = x;
        mLastY = y;
        return super.dispatchTouchEvent(event);
    }
```

父View需要重写onInterceptTouchEvent方法：



```java
    public boolean onInterceptTouchEvent(MotionEvent event) {

        int action = event.getAction();
        if (action == MotionEvent.ACTION_DOWN) {
            return false;
        } else {
            return true;
        }
    }
```

> **使用内部拦截法需要注意：**
>
> - 内部拦截法要求父View不能拦截ACTION_DOWN事件，由于ACTION_DOWN不受FLAG_DISALLOW_INTERCEPT标志位控制，一旦父容器拦截ACTION_DOWN那么所有的事件都不会传递给子View。
> - 滑动策略的逻辑放在子View的dispatchTouchEvent方法的ACTION_MOVE中，如果父容器需要获取点击事件则调用 parent.requestDisallowInterceptTouchEvent(false)方法，让父容器去拦截事件。

## 6.5 滑动冲突解决示例代码

------

理论最终的落脚是在实践，下面我通过一个例子来演示外部解决法和内部解决法解决滑动冲突，大家只要get到了精髓，那么今后遇到滑动冲突问题都将迎刃而解，不再是开发拦路虎！

我们一开始说过ViewPager已经默认给我们处理了滑动冲突，而它作为ViewGroup使用的是外部拦截法解决的冲突，即在onInterceptTouchEvent方法中进行判断的。为了造成滑动冲突场景，那么我们自定义一个ViewPager，重写onInterceptTouchEvent方法并默认返回false，如下所示：



```java
public class BadViewPager extends ViewPager {
    public BadViewPager(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return false;
    }
}
```

这样，一个好好的ViewPager就被我们玩坏了！





接下来新建一个ScrollConflicActivity用来测试滑动冲突。



```java
public class ScrollConflictActivity extends BaseActivity {
    private BadViewPager mViewPager;
    private List<View> mViews;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_scroll_conflict);
        initViews();
        initData(false);
    }

    protected void initViews() {
        mViewPager = findAviewById(R.id.viewpager);
        mViews = new ArrayList<>();
    }

    protected void initData(final boolean isListView) {
        //初始化mViews列表
        Flowable.just("view1", "view2", "view3", "view4").subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                //当前View
                View view;
                if (isListView) {
                    //初始化ListView
                    ListView listView = new ListView(mContext);
                    final ArrayList<String> datas = new ArrayList<>();
                    //初始化数据，datas ,data0 ...data49
                    Flowable.range(0, 50).subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            datas.add("data" + integer);
                        }
                    });
                    //初始化adapter
                    ArrayAdapter<String> adapter = new ArrayAdapter<>
                            (mContext, android.R.layout.simple_list_item_1, datas);
                    //设置adapter
                    listView.setAdapter(adapter);
                    //将ListView赋值给当前View
                    view = listView;
                } else {
                    //初始化TextView
                    TextView textView = new TextView(mContext);
                    textView.setGravity(Gravity.CENTER);
                    textView.setText(s);
                    //将TextView赋值给当前View
                    view = textView;
                }
                //将当前View添加到ViewPager的ViewList中去
                mViews.add(view);
            }
        });
        //设置ViewPager的Adapter
        mViewPager.setAdapter(new BasePagerAdapter<>(mViews));
    }
}
```

**注：Flowable是RxJava2的方法，这里其实用for循环也是一样的
 BasePagerAdapter是[BaseProject](https://www.jianshu.com/p/d5ad3f127ebf)里的方法**

上面的代码我们使用了BadViewPager，初始化了BadViewPager里面的子View。
 initData(false);方法传false表示里面的子View是一个TextView，传true表示里面的子View是ListView。首先我们看BadViewPager里面子View是TextView是否可以滑动。

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210626203305)



BadViewPager_bad_textview.gif

似乎对BadViewPager的滑动没有任何影响。



大家别急，我们来分析一下，BadViewPager的onInterceptTouchEvent默认返回false则所有事件都会给子View去处理。大家是否还记得上一篇说到的View处理事件的原则？

**View 的onTouchEvent 方法默认都会消费掉事件（返回true），除非它是不可点击的（clickable和longClickable同时为false），View的longClickable默认为false，clickable需要区分情况，如Button的clickable默认为true，而TextView的clickable默认为false。**

所以TextView默认并没有消费事件，因为他是不可点击的。事件会交由父View即BadViewPager的onTouchEvent方法去处理。所以它自然是可以滑动的。

我们将textview的Clickable设置成true，即让它来消费事件。大家再看看呢

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210626203326)

BadViewPager_bad_textview_clickable.gif

所以我们不难推测如果将TextView换成Button，将是一样的无法滑动的效果。虽然这并不是常规的滑动冲突（子View不是滑动的），但是造成的原因其实是一样的，没有做滑动判断导致父View不能正确响应滑动事件。

接下来稍稍修改一下代码 initData(true);传入true，即BadViewPager的子View使用ListView，显然ListView是可以滑动的，BadViewPager是不能滑动的。我们分别通过外部拦截和内部拦截方法来对BadViewPager进行修复。

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210626203351)

BadViewPager_bad_listview.gif

### 6.5.1 1.外部拦截法Fix BadViewPager：



```java
public class BadViewPager extends ViewPager {
    private static final String TAG = "BadViewPager";

    private int mLastXIntercept;
    private int mLastYIntercept;

    public BadViewPager(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        boolean intercepted = false;
        int x = (int) ev.getX();
        int y = (int) ev.getY();
        final int action = ev.getAction() & MotionEvent.ACTION_MASK;
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                intercepted = false;
                //调用ViewPager的onInterceptTouchEvent方法初始化mActivePointerId
                super.onInterceptTouchEvent(ev);
                break;
            case MotionEvent.ACTION_MOVE:
                //横坐标位移增量
                int deltaX = x - mLastXIntercept;
                //纵坐标位移增量
                int deltaY = y - mLastYIntercept;
                if (Math.abs(deltaX)>Math.abs(deltaY)){
                    intercepted = true;
                }else{
                    intercepted = false;
                }
                break;
            case MotionEvent.ACTION_UP:
                intercepted = false;
                break;
            default:
                break;
        }

        mLastXIntercept = x;
        mLastYIntercept = y;

        LogUtil.e(TAG,"intercepted = "+intercepted);
        return intercepted;
    }
}
```

根据我们的外部拦截法的套路，需要重写BadViewPager 的onInterceptTouchEvent方法，并且ACTION_DOWN 和 ACTION_UP返回为false。处理逻辑在ACTION_MOVE中，Math.abs(deltaX)>Math.abs(deltaY)表示横向位移增量大于竖向位移增量，即水平滑动，则BadViewPager 拦截事件。

这里我们在ACTION_DOWN 当中还调用了super.onInterceptTouchEvent(ev);即ViewPager的onInterceptTouchEvent方法。主要是为了初始化ViewPager的成员变量mActivePointerId。mActivePointerId默认值为-1，在ViewPager的onTouchEvent方法的ACTION_MOVE中有这么一段：



```java
class ViewPager
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        ...
        switch (action & MotionEventCompat.ACTION_MASK) {
            case MotionEvent.ACTION_MOVE:
                if (!mIsBeingDragged) {
                    final int pointerIndex = ev.findPointerIndex(mActivePointerId);
                    if (pointerIndex == -1) {
                        // A child has consumed some touch events and put us into an inconsistent
                        // state.
                        needsInvalidate = resetTouch();
                        break;
                    }
                    //具体的滑动操作...
                }
                ...
                break;
                ...
        }
        ...
    }
```

假如mActivePointerId不进行初始化，ViewPager会认为这个事件已经被子View给消费了，然后break掉，接下来的滑动操作也就不执行了。

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210626203412)

BadViewPager_fixed_listview.gif

### 6.5.2 2.内部拦截法Fix BadViewPager：

内部拦截法需要重写ListView的dispatchTouchEvent方法，所以我们自定义一个ListView：



```java
public class FixListView extends ListView {
    private static final String TAG = "FixListView";

    private int mLastX;
    private int mLastY;

    public FixListView(Context context) {
        super(context);
    }

    public FixListView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        int x = (int) ev.getX();
        int y = (int) ev.getY();

        final int action = ev.getAction() & MotionEvent.ACTION_MASK;
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                getParent().requestDisallowInterceptTouchEvent(true);
                break;
            case MotionEvent.ACTION_MOVE:
                //水平移动的增量
                int deltaX = x - mLastX;
                //竖直移动的增量
                int deltaY = y - mLastY;
                //当水平增量大于竖直增量时，表示水平滑动，此时需要父View去处理事件
                if (Math.abs(deltaX) > Math.abs(deltaY)){
                    getParent().requestDisallowInterceptTouchEvent(false);
                }
                break;
            case MotionEvent.ACTION_UP:
                break;
            default:
                break;
        }
        mLastX = x;
        mLastY = y;
        return super.dispatchTouchEvent(ev);
    }
}
```

再看BadViewPager，需要重写拦截方法



```java
public class BadViewPager extends ViewPager {
    private static final String TAG = "BadViewPager";

    public BadViewPager(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        final int action = ev.getAction() & MotionEvent.ACTION_MASK;
        if (action == MotionEvent.ACTION_DOWN){
            super.onInterceptTouchEvent(ev);
            return false;
        }
        return true;
    }
}
```

可以看到和我们的套路代码基本上一样，只是ACTION_MOVE中有我们自己的逻辑处理，处理的方式与外部拦截法也是一致的，并且效果也一样，在此不作赘述。

**大家只用掌握上述滑动冲突的解决套路，不论场景是不同方向，还是同方向，还是乱七八糟的堆加在一起，就用套路去解决，万变不离其宗。根据上述的外部拦截和内部拦截法，可以看出外部拦截法实现起来更加简单，而且也符合View的正常事件分发机制，所以推荐使用外部拦截法（重写父View的onInterceptTouchEvent，父View决定是否拦截）来处理滑动冲突**

# 7 ACTION_MOVE 和 ACTION_UP
https://www.jianshu.com/p/e99b5e8bd67b

上面讲解的都是针对ACTION_DOWN的事件传递，ACTION_MOVE和ACTION_UP在传递的过程中并不是和ACTION_DOWN 一样，你在执行ACTION_DOWN的时候返回了false，后面一系列其它的action就不会再得到执行了。简单的说，就是**当dispatchTouchEvent在进行事件分发的时候，只有前一个事件（如ACTION_DOWN）返回true，才会收到ACTION_MOVE和ACTION_UP的事件**。具体这句话很多博客都说了，但是具体含义是什么呢？我们来看一下下面的具体分析。

上面提到过了，事件如果不被打断的话是会不断往下传到叶子层（View），然后又不断回传到Activity，dispatchTouchEvent 和 onTouchEvent 可以通过return true 消费事件，终结事件传递，而onInterceptTouchEvent 并不能消费事件，它相当于是一个分叉口起到分流导流的作用，ACTION_MOVE和ACTION_UP 会在哪些函数被调用，之前说了并不是哪个函数收到了ACTION_DOWN，就会收到 ACTION_MOVE 等后续的事件的。  
下面通过几张图看看不同场景下，ACTION_MOVE事件和ACTION_UP事件的具体走向并总结一下规律。

**1、我们在ViewGroup1 的dispatchTouchEvent 方法返回true消费这次事件**

ACTION_DOWN 事件从（Activity的dispatchTouchEvent）--------> （ViewGroup1 的dispatchTouchEvent） 后结束传递，事件被消费（如下图红色的箭头代码ACTION_DOWN 事件的流向）。

```java
//打印日志
Activity | dispatchTouchEvent --> ACTION_DOWN 
ViewGroup1 | dispatchTouchEvent --> ACTION_DOWN
---->消费
```

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132241684.png)


在这种场景下ACTION_MOVE和ACTION_UP 将如何呢，看下面的打出来的日志

```rust
Activity | dispatchTouchEvent --> ACTION_MOVE 
ViewGroup1 | dispatchTouchEvent --> ACTION_MOVE
----
TouchEventActivity | dispatchTouchEvent --> ACTION_UP 
ViewGroup1 | dispatchTouchEvent --> ACTION_UP
----
```

下图中  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向

  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132241836.png)


**2、我们在ViewGroup2 的dispatchTouchEvent 返回true消费这次事件**

```rust
Activity | dispatchTouchEvent --> ACTION_DOWN 
ViewGroup1 | dispatchTouchEvent --> ACTION_DOWN
ViewGroup1 | onInterceptTouchEvent --> ACTION_DOWN
ViewGroup2 | dispatchTouchEvent --> ACTION_DOWN
---->消费
Activity | dispatchTouchEvent --> ACTION_MOVE 
ViewGroup1 | dispatchTouchEvent --> ACTION_MOVE
ViewGroup1 | onInterceptTouchEvent --> ACTION_MOVE
ViewGroup2 | dispatchTouchEvent --> ACTION_MOVE
----
TouchEventActivity | dispatchTouchEvent --> ACTION_UP 
ViewGroup1 | dispatchTouchEvent --> ACTION_UP
ViewGroup1 | onInterceptTouchEvent --> ACTION_UP
ViewGroup2 | dispatchTouchEvent --> ACTION_UP
----
```

红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向

  
![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132241802.png)



**3、我们在View 的dispatchTouchEvent 返回true消费这次事件**  
这个我不就画图了，效果和在ViewGroup2 的dispatchTouchEvent return true的差不多，同样的收到ACTION_DOWN 的dispatchTouchEvent函数都能收到 ACTION_MOVE和ACTION_UP。  
**所以我们就基本可以得出结论如果在某个控件的dispatchTouchEvent 返回true消费终结事件，那么收到ACTION_DOWN 的函数也能收到 ACTION_MOVE和ACTION_UP。**

**4、我们在View 的onTouchEvent 返回true消费这次事件**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132242287.png)


**5、我们在ViewGroup 2 的onTouchEvent 返回true消费这次事件**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132242435.png)
  
**6、我们在ViewGroup 1 的onTouchEvent 返回true消费这次事件**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132242225.png)

**7、我们在Activity 的onTouchEvent 返回true消费这次事件**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132243108.png)


**8、我们在View的dispatchTouchEvent 返回false并且Activity 的onTouchEvent 返回true消费这次事件**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132243903.png)


**9、我们在View的dispatchTouchEvent 返回false并且ViewGroup 1 的onTouchEvent 返回true消费这次事件**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132243788.png)


**10、我们在View的dispatchTouchEvent 返回false并且在ViewGroup 2 的onTouchEvent 返回true消费这次事件**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132243101.png)


**11、我们在ViewGroup2的dispatchTouchEvent 返回false并且在ViewGroup1 的onTouchEvent返回true消费这次事件**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132243190.png)


**12、我们在ViewGroup2的onInterceptTouchEvent 返回true拦截此次事件并且在ViewGroup 1 的onTouchEvent返回true消费这次事件。**  
红色的箭头代表ACTION_DOWN 事件的流向  
蓝色的箭头代表ACTION_MOVE 和 ACTION_UP 事件的流向  

![image.png](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202304132244350.png)


一下子画了好多图，还有好几种情况就不再画了，相信你也看出规律了，对于在onTouchEvent消费事件的情况：**在哪个View的onTouchEvent 返回true，那么ACTION_MOVE和ACTION_UP的事件从上往下传到这个View后就不再往下传递了，而直接传给自己的onTouchEvent 并结束本次事件传递过程。**

对于ACTION_MOVE、ACTION_UP总结：**ACTION_DOWN事件在哪个控件消费了（return true）， 那么ACTION_MOVE和ACTION_UP就会从上往下（通过dispatchTouchEvent）做事件分发往下传，就只会传到这个控件，不会继续往下传，如果ACTION_DOWN事件是在dispatchTouchEvent消费，那么事件到此为止停止传递，如果ACTION_DOWN事件是在onTouchEvent消费的，那么会把ACTION_MOVE或ACTION_UP事件传给该控件的onTouchEvent处理并结束传递。**

tips : 最近刚做了一个自定义控件，里面做了不少事件分发的处理和交互，个人是觉得可以当做本篇文章的一个实践，大家可以看下源码事件分发相关的的部分代码，可能更有体会，链接地址：[CalendarListView](https://link.jianshu.com?t=https://github.com/Kelin-Hong/CalendarListView)**


# 8 Action.Cancel什么时候会出现
https://juejin.cn/post/7004794729237856287

1. 如果在父View中拦截ACTION_UP或ACTION_MOVE，在第一次父视图拦截消息的瞬间，父视图指定子视图不 接受后续消息了，同时子视图会收到ACTION_CANCEL事件。 

2. 如果触摸某个控件，但是又不是在这个控件的区域上 抬起(移动到别的地方了)，就会出现action_cancel。


-   ACTION_CANCEL的触发时机
-   滑出子View区域会发生什么？为什么不响应`onClick()`事件

首先看一下官方的解释：

```java
/**
 * Constant for {@link #getActionMasked}: The current gesture has been aborted.
 * You will not receive any more points in it.  You should treat this as
 * an up event, but not perform any action that you normally would.
 */
public static final int ACTION_CANCEL           = 3;
复制代码
```

说人话就是：当前的手势被中止了，你不会再收到任何事件了，你可以把它当做一个ACTION_UP事件，但是不要执行正常情况下的逻辑。

## 8.1 ACTION_CANCEL的触发时机

有四种情况会触发`ACTION_CANCEL`:

-   在子View处理事件的过程中，父View对事件拦截
-   ACTION_DOWN初始化操作
-   在子View处理事件的过程中被从父View中移除时
-   子View被设置了`PFLAG_CANCEL_NEXT_UP_EVENT`标记时

### 8.1.1 1父view拦截事件

首先要了解ViewGroup什么情况下会拦截事件，Look the Fuck Resource Code：

```java
/**
 * {@inheritDoc}
 */
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	...

    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
		...
        // Check for interception.
        final boolean intercepted;
        // 判断条件一
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            // 判断条件二
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }
        ...
    }
    ...
}

```

有两个条件

-   MotionEvent.ACTION_DOWN事件或者mFirstTouchTarget非空也就是有子view在处理事件
-   子view没有做拦截，也就是没有调用`ViewParent#requestDisallowInterceptTouchEvent(true)`

如果满足上面的两个条件才会执行`onInterceptTouchEvent(ev)`。

如果ViewGroup拦截了事件，则`intercepted`变量为true，接着往下看：

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        ...

        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                // 当mFirstTouchTarget != null，也就是子view处理了事件
                // 此时如果父ViewGroup拦截了事件，intercepted==true
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }

        ...

        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            ...
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    ...
                } else {
                    // 判断一：此时cancelChild == true
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;

					// 判断二：给child发送cancel事件
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    ...
                }
                ...
            }
        }
        ...
    }
    ...
    return handled;
}

```

以上判断一处`cancelChild`为true，然后进入判断二中一看究竟：

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
    final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        // 将event设置成ACTION_CANCEL
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            ...
        } else {
            // 分发给child
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    ...
}

```

当参数`cancel`为ture时会将`event`设置为`MotionEvent.ACTION_CANCEL`，然后分发给child。

### 8.1.2 2ACTION_DOWN初始化操作

```java
public boolean dispatchTouchEvent(MotionEvent ev) {

    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            // 取消并清除所有的Touch目标
            cancelAndClearTouchTargets(ev);
            resetTouchState();
    	}
    	...
    }
    ...
}

```

系统可能会由于App切换、ANR等原因丢失了up，cancel事件。

因此需要在`ACTION_DOWN`时丢弃掉所有前面的状态，具体代码如下：

```java
private void cancelAndClearTouchTargets(MotionEvent event) {
    if (mFirstTouchTarget != null) {
        boolean syntheticEvent = false;
        if (event == null) {
            final long now = SystemClock.uptimeMillis();
            event = MotionEvent.obtain(now, now,
                    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
            syntheticEvent = true;
        }

        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            resetCancelNextUpFlag(target.child);
            // 分发事件同情况一
            dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
        }
        ...
    }
}

```

PS：在`dispatchDetachedFromWindow()`中也会调用`cancelAndClearTouchTargets()`

### 8.1.3 3在子View处理事件的过程中被从父View中移除时

```java
public void removeView(View view) {
    if (removeViewInternal(view)) {
        requestLayout();
        invalidate(true);
    }
}

private boolean removeViewInternal(View view) {
    final int index = indexOfChild(view);
    if (index >= 0) {
        removeViewInternal(index, view);
        return true;
    }
    return false;
}

private void removeViewInternal(int index, View view) {

    ...
    cancelTouchTarget(view);
	...
}

private void cancelTouchTarget(View view) {
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    while (target != null) {
        final TouchTarget next = target.next;
        if (target.child == view) {
            ...
            // 创建ACTION_CANCEL事件
            MotionEvent event = MotionEvent.obtain(now, now,
                    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
            分发给目标view
            view.dispatchTouchEvent(event);
            event.recycle();
            return;
        }
        predecessor = target;
        target = next;
    }
}

```

### 8.1.4 4子View被设置了`PFLAG_CANCEL_NEXT_UP_EVENT`标记时

在情况一种的两个判断处：

```java
// 判断一：此时cancelChild == true
final boolean cancelChild = resetCancelNextUpFlag(target.child)
|| intercepted;

// 判断二：给child发送cancel事件
if (dispatchTransformedTouchEvent(ev, cancelChild,
    target.child, target.pointerIdBits)) {
    handled = true;
}

```

当`resetCancelNextUpFlag(target.child)`为true时同样也会导致cancel，查看代码：

```java
/**
 * Indicates whether the view is temporarily detached.
 *
 * @hide
 */
static final int PFLAG_CANCEL_NEXT_UP_EVENT        = 0x04000000;

private static boolean resetCancelNextUpFlag(View view) {
    if ((view.mPrivateFlags & PFLAG_CANCEL_NEXT_UP_EVENT) != 0) {
        view.mPrivateFlags &= ~PFLAG_CANCEL_NEXT_UP_EVENT;
        return true;
    }
    return false;
}
复制代码
```

根据注释大概意思是，该view暂时`detached`，`detached`是什么意思？就是和`attached`相反的那个，具体什么时候打了这个标记，我觉得没必要深究。

以上四种情况最重要的就是第一种，后面的只需了解即可。

## 8.2 滑出子View区域会发生什么？

了解了什么情况下会触发`ACTION_CANCEL`，那么针对问题：滑出子View区域会触发`ACTION_CANCEL`吗？这个问题就很明确了：不会。

实践是检验真理的唯一标准，代码撸起来：

```java
public class MyButton extends androidx.appcompat.widget.AppCompatButton {

	@Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                LogUtil.d("ACTION_DOWN");
                break;
            case MotionEvent.ACTION_MOVE:
                LogUtil.d("ACTION_MOVE");
                break;
            case MotionEvent.ACTION_UP:
                LogUtil.d("ACTION_UP");
                break;
            case MotionEvent.ACTION_CANCEL:
                LogUtil.d("ACTION_CANCEL");
                break;
        }
        return super.onTouchEvent(event);
    }
}
复制代码
```

一波操作以后日志如下：

> (MyButton.java:32) -->ACTION_DOWN (MyButton.java:36) -->ACTION_MOVE (MyButton.java:36) -->ACTION_MOVE (MyButton.java:36) -->ACTION_MOVE (MyButton.java:36) -->ACTION_MOVE (MyButton.java:36) -->ACTION_MOVE (MyButton.java:39) -->ACTION_UP

滑出view后依然可以收到`ACTION_MOVE`和`ACTION_UP`事件。

为什么有人会认为滑出view后会收到`ACTION_CANCEL`呢？

我想是因为滑出view后，view的`onClick()`不会触发了，所以有人就以为是触发了`ACTION_CANCEL`。

那么为什么滑出view后不会触发`onClick`呢？再来看看View的源码：

在view的`onTouchEvent()`中：

```java
case MotionEvent.ACTION_MOVE:
    // Be lenient about moving outside of buttons
	// 判断是否超出view的边界
    if (!pointInView(x, y, mTouchSlop)) {
        // Outside button
        if ((mPrivateFlags & PRESSED) != 0) {
            // 这里改变状态为 not PRESSED
            // Need to switch from pressed to not pressed
            mPrivateFlags &= ~PRESSED;
        }
    }
    break;
    
case MotionEvent.ACTION_UP:
    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
    // 可以看到当move出view范围后，这里走不进去了
    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
        ...
        performClick();
        ...
    }
    mIgnoreNextUpEvent = false;
    break;
复制代码
```

1，在`ACTION_MOVE`中会判断事件的位置是否超出view的边界，如果超出边界则将`mPrivateFlags`置为`not PRESSED`状态。

2，在`ACTION_UP`中判断只有当`mPrivateFlags`包含`PRESSED`状态时才会执行`performClick()`等。

因此滑出view后不会执行`onClick()`。

##### 8.2.1.1.1 结论：

-   滑出view范围后，如果父view没有拦截事件，则会继续受到`ACTION_MOVE`和`ACTION_UP`等事件。
-   一旦滑出view范围，view会被移除`PRESSED`标记，这个是不可逆的，然后在`ACTION_UP`中不会执行`performClick()`等逻辑。

  

作者：giswangsj  
链接：https://juejin.cn/post/7004794729237856287  
来源：稀土掘金  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。




# 9 问题

## 9.1 onTouchListener onTouchEvent onClick的执行顺序



## 9.2 怎么拦截事件 onTouchEvent如果返回false onClick还会执行么？



## 9.3 View中onTouch，onTouchEvent和onClick的执行顺序


## 9.4 Android事件分发机制一：事件是如何到达activity的？ : 从window机制出发分析了事件分发的整体流程，以及事件分发的真正起点

## 9.5 Android事件分发机制二：viewGroup与view对事件的处理 : 源码分析了viewGroup和view是如何分发事件的

## 9.6 Android事件分发机制三：事件分发工作流程 : 分析了触摸事件在控件树中的分发流程模型

## 9.7 Android事件分发机制四：学了事件分发有什么用？ : 从实战的角度剖析事件分发的运用


