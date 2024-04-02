# MVC

- Model：模型层，数据相关的操作
- View：视图层，用户界面渲染逻辑
- Controller：控制器，数据模型和视图之间通信的桥梁

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210823114612.jpg)



Model-View-Controller，经典模式，很容易理解，主要缺点有两个：

1. View对Model的依赖，会导致View也包含了业务逻辑；
2. Controller会变得很厚很复杂。


架构介绍
​​Model​​：数据模型，负责处理数据的加载或存储，比如我们从数据库或者网络获取数据
​​View​​：视图，负责界面数据的展示，与用户进行交互，也就是我们的xml布局文件
​​Controller​​：控制器，负责逻辑业务的处理，也就是我们的Activity



## MVC 优点

1.​​View​​​ 接受用户的请求，然后将请求传递给​​Controller​​
2.​​Controller​​​ 进行业务逻辑处理后，通知​​Model​​去更新
​​Model​​​ 数据更新后，通知​​View​​ 去更新界面显示
这里容易发生耦合
​​View --> Controller​​​，也就是反应​​View​​​ 的一些用户事件（点击触摸事件）到​​Activity上​​
​​Controller --> Model​​​, 也就是​​Activity​​ 去读写一些我们需要的数据
​​Controller --> View​​​, 也就是​​Activity​​​ 在获取数据之后，将更新内容反映到​​View​​上`


## MVC 缺点
Android中使用了​​Activity​​​ 来充当​​Controller​​​，但实际上一些​​UI​​​ 也是由​​Activity​​​ 来控制的，比如进度条等。因此部分视图就会跟​​Controller​​​ 捆绑在同一个类了。同时，由于​​Activity​​​ 的职责过大，​​Activity​​ 类的代码也会迅速膨胀
主要表现就是我们的​​Activity​​​ 太重了，经常一写就是几百上千行了。造成这种问题的原因就是​​Controller​​​ 层和​​View​​​ 层的关系太过紧密，也就是​​Activity​​​ 中有太多操作​​View​​ 的代码了
MVC还有一个重要的缺陷就是​​View​​​ 跟​​Model​​ 是有交互的，没有做到完全的分离，这就会产生耦合。

PS: 但是！但是！其实Android这种并称不上传统的MVC结构，因为Activity又可以叫View层又可以叫Controller层，所以我觉得这种Android默认的开发结构，其实称不上什么MVC项目架构，因为他本身就是Android一开始默认的开发形式，所有东西都往Activity中丢，然后能封装的封装一下，根本分不出来这些层




# MVP

Model-View-Presenter，MVC的一个演变模式，将Controller换成了**Presenter**，主要为了解决**上述第一个缺点，将View和Model解耦**，不过第二个缺点依然没有解决。


MVP 是 Model View Presenter 的缩写，可以说是 MVC 模式的改良，相对于 MVC 有了各层负责的任务和数据流动方式都有了部分变化

- Model：和具体业务无关的数据处理
- View：用户界面渲染逻辑
- Presenter：响应视图指令，同时进行相关业务处理，必要时候获调用 Model 获取底层数据，返回指令结果到视图，驱动视图渲染



![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210823114703.jpg)



Model​​：**数据模型**，比如我们从数据库或者网络获取数据
跟​​MVC​​​不同的地方在于​​Model​​​ 不会跟​​View​​​ 发生交互，只会跟​​Presenter​​ 交互
​​View​​：**视图**，也就是我们的xml布局文件和Activity
MVP在实现上来说可以有多种思路，不同的实现方式其优缺点也是不同的，具体问题具体分析
eg: 也有场景下​​Activity​​​ 作为​​Presenter​​ 会更好一些
​​Presenter​​：主持人，单独的类，只做调度工作。
​​View --> Presenter​​​，反应​​View​​​ 的一些用户事件到​​Presenter​​上。
​​Presenter --> Model​​​,​​Presenter​​去读写操作一些我们需要的数据。
​​Presenter--> View​​​,​​Presenter​​​在获取数据之后，将更新内容反馈给​​Activity​​​，进行​​view​​ 更新。
通常View与Presenter是一对一的，但复杂的View可以绑定多个Presenter来处理逻辑。

## 优势
这种的优点就是确实大大减少了 ​​Activity​​​ 的负担，让 ​​Activity​​​ 主要承担一个更新 ​​View​​​ 的工作，然后把跟 ​​Model​​​ 交互的工作转移给了 ​​Presenter​​​，从而由​​Presenter​​​方来控制和交互 ​​Model​​​方以及 ​​View​​。所以让项目更加明确简单，顺序性思维开发。

​​View​​​ 与​​Model​​ 完全分离，我们可以修改视图而不影响模型。
可以更高效地使用模型，因为所有的交互都发生​​Presenter​​ 中
​​Presenter​​​ 与​​View​​ 的交互是通过接口来进行的，有利于添加**单元测试**。


## 劣势
缺点也很明显：
1. 首先就是代码量大大增加了，每个页面或者说功能点，都要专门写一个​​Presenter​​类，并且由于是面向接口编程，需要增加大量接口，会有大量繁琐的回调。页面逻辑复杂的话，相应的接口也会变多，增加维护成本
2. 其次，由于​​Presenter​​​ 里持有了​​Activity​​​ 对象，所以可能会导致内存泄漏或者​​view​​ 空指针，这也是需要注意的地方。
3. 系统内存不足时，系统会回收​​Activity​​​。一般我们都是用​​OnSaveInstanceState()​​​ 去保存状态，用​​OnRestoreInstanceState()​​​ 去恢复状态。但是在我们的​​MVP​​​中，​​View​​​层是不应该去直接操作​​Model​​​的，所以这样做不合理，同时也增大了​​M​​​ 与​​V​​的耦合。
    解决办法是不要将​​Activity​​​ 作为​​View​​​ 层，可以把​​Activity​​​当​​Presenter​​来处理。具体实现这里就不分析了，有兴趣的可以研究一下
​​ 4.UI​​​ 改变的话，比如​​TextView​​​ 替换​​EditText​​​，可能导致​​Presente​​​ 的一些更新​​UI​​ 的接口也跟着需要更改，存在一定的耦合


## 对应的场景





# MVVM
https://juejin.cn/post/7083137114648346661#heading-13

https://www.jianshu.com/p/54f82b17c4d3

https://blog.csdn.net/u011033906/article/details/118113466

Model-View-ViewModel，是对MVP的一个优化模式，采用了双向绑定：View的变动，自动反映在ViewModel，反之亦然。

架构模式上，我不会推崇说哪种模式好，每种模式都各有优点，也各有极限性。越高级的模式复杂性越高，实现起来也越难。最近火热的微服务架构，比起MVC，复杂度不知增加了多少倍。


MVVM 可以写成 MV-VM，是 Model View - ViewModel 的缩写，可以算是 MVP 模式的变种，View 和 Model 职责和 MVP 相同，但 ViewModel 主要靠 DataBinding 把 View 和 Model 做了自动关联，框架替应用开发者实现数据变化后的视图更新，相当于简化了 Presenter 的部分功能

![img](https://cdn.jsdelivr.net/gh/wp3355168/Typora-Picgo-Gitee/img/20210823115232.jpg)







有 ​​Google​​​ 官方加持，更新了 ​​**Jetpack**​​​ 中很多架构组件，比如 ​​ViewModel​​​，​​Livedata​​​，​​DataBinding​​等等，所以这个是现在的主流框架和官方推崇的框架。

Model：数据模型，比如我们从数据库或者网络获取数据。View：视图，也就是我们的xml布局文件和Activity。ViewModel：关联层，将Model和View绑定，使他们之间可以相互绑定实时更新

​​Model​​：模型层，负责处理数据的加载或存储。与 MVP中的M一样。
​​View​​：视图层，负责界面数据的展示，与用户进行交互。与MVP中的V一样。
​​ViewModel​​​：视图模型，负责完成​​View​​​ 于​​Model​​ 间的交互,负责业务逻辑。
​​View​​​ 与​​ViewModel​​ 进行绑定，能够实现双向的交互
​​ViewModel​​​数据改变时，​​View​​​ 会相应变动​​UI​​，反之亦然
​​ViewModel​​​ 进行业务逻辑处理，通知​​Model​​ 去更新
可以与​​View​​​ 实现​​databinding​​ 双向绑定 ， 使他们之间可以相互绑定实时更新
​​Model​​​ 数据更新后，把新数据传递给​​ViewModel​​

eg:​​Activity​​​ 中监听​​viewModel.value​​​ 变化，监听到之后改变​​view​​ 的值
​​View --> ViewModel -->View​​，双向绑定，数据改动可以反映到界面，界面的修改可以反映到数据
​​ViewModel --> Model​​, 操作一些我们需要的数据

## 优点
​​MVVM​​​ 已经被实践证明是一种优秀的设计模式。能很好地将 ​​UI 、交互逻辑、业务逻辑​​​ 和 ​​数据​​ 解耦

降低​​View​​​和​​Controller​​ 的耦合，减轻了视图的压力
Activity 代码不会像 MVC 那样那么臃肿，方便维护
相比于​​MVP​​​ 中​​Presente​​​ 与​​View​​​ 存在耦合。​​ViewModel​​​ 与​​View​​​ 的耦合则更低，​​ViewModel​​ 只负责处理和提供数据，UI的改变，比如TextView 替换 EditText，ViewModel 几乎不需要更改任何代码，只需专注于数据处理就可以了。
​​ViewModel​​ 里面只包含数据和业务逻辑，没有UI的东西，方便单元测试。


## 缺点
数据绑定使得程序较难调试
界面出现异常时，有可能是 View 的代码有问题，也可能是 Model 的代码有问题。
由于数据绑定使得数据能够快速传递到其他位置，因此要定位出异常就比较有难度了。

## Why use Jetpack + MVVM？

我们通常使用 Jetpack 配合 MVVM 一起使用，为什么呢？

- 快速开发
组件可以单独采用（不过这些组件是为协同工作而构建的），同时利用 Kotlin 语言功能帮助您提高工作效率。
- 消除样板代码
Android Jetpack 可管理繁琐的 Activity（如后台任务、导航和生命周期管理），以便您可以专注于如何让自己的应用出类拔萃。
- 构建高质量的强大应用
Android Jetpack 组件围绕现代化设计实践构建而成，具有向后兼容性，可以减少崩溃和内存泄漏。
开发者可以减少许多样板代码的书写，只需要通过模版工具自动生成就可以了，在取缔非常多的耗时的重复工作的同时，减少了很多因为忘记 unRegister带来的各种问题




# 参考

https://zhuanlan.zhihu.com/p/165572019
