https://blog.csdn.net/u011578734/article/details/108339776

一步一步教你如何在AndroidStudio查看Android源码（AOSP源码）
# idegen工具
要将Android系统源代码工程导入到Eclipse或者IntelliJ IDEA，关键是要有相应的工程配置文件。

idegen专门为IDE环境调试源码而设计的工具，idegen可以用来生成针对Eclipse和IntelliJ IDEA的Android系统源代码工程配置文件，它位于Android系统源代码工程目录的下列位置：

development/tools/idegen/


# AS中导入AOSP源码
将工程导入AS需要下面三个步骤：

获取到idegen.jar
获取idegen.sh 执行生成android.ipr/android.iml
Android sutdio 选择android.ipr导入
生成android.ipr等文件
执行下面的命令即可生成android.ipr等文件：

cd ～/aosp //具体的源码根目录
source build/envsetup.sh //用于初始化环境变量
mmm development/tools/idegen/  //生成文件out/host/linux-x86/framework/idegen.jar
./development/tools/idegen/idegen.sh//源码根目录生成文件android.ipr(工程相关设置), android.iml(模块相关配置)
1
2
3
4
注意：如果是mac，执行命令之前，首先要进入bash，方法也很简单：

bash
1
否则会报错：报错Couldn’t find directory development/tools/idegen/

导入AndroidStudio
打开AS，点击File -> Open，选中前面生成的android.ipr文件即可，该过程比较耗时。

导入AS配置优化
如果直接进行导入，导入之后，可以看到Android Studio下方，Indexing…会一直显示，时间非常长。其实我们大多数情况下只需要framework下的代码，可以进行一些排除操作。

android.iml文件

iml文件是idea组织工程的文件, 里面记录了各种记录模块, 文件夹以及依赖的信息。一般而言, 创建的工程都会有这个文件, 它的本质是一个工程组织文件, 和Maven的pom.xml, gradle的build.gradle, 等组织工程和处理依赖关系的文件并没有什么差别。

打开android.iml文件，我们会发现这个而文件配置项非常多，主要有类标签：

sourceFolder：表示包含的文件目录，通常我们只需要留下framewrok即可。
excludeFolder：exclude顾名思义就是不包含的意思。我们有很多目录直接就不想让Studio去管它，不管是索引还是什么等等，所以只需要将这些目录配置到中就好了。
android.iml文件修改

打开android.iml文件，那么我们可以有选择的导入如下：

  <component name="NewModuleRootManager" inherit-compiler-output="true">
    <exclude-output />
    <content url="file://$MODULE_DIR$">
      <sourceFolder url="file://$MODULE_DIR$/../10.0.0_r2frameworks/base/core/java" type="kotlin-source" />
      <excludeFolder url="file://$MODULE_DIR$/.repo" />
      <excludeFolder url="file://$MODULE_DIR$/external/bluetooth" />
      <excludeFolder url="file://$MODULE_DIR$/external/chromium" />
      <excludeFolder url="file://$MODULE_DIR$/external/icu4c" />
      <excludeFolder url="file://$MODULE_DIR$/external/webkit" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/docs" />
      <excludeFolder url="file://$MODULE_DIR$/out/eclipse" />
      <excludeFolder url="file://$MODULE_DIR$/out/host" />
      <excludeFolder url="file://$MODULE_DIR$/out/target/common/docs" />
      <excludeFolder url="file://$MODULE_DIR$/out/target/common/obj/JAVA_LIBRARIES/android_stubs_current_intermediates" />
      <excludeFolder url="file://$MODULE_DIR$/out/target/product" />
      <excludeFolder url="file://$MODULE_DIR$/prebuilt" />
      <excludeFolder url="file://$MODULE_DIR$/../10.0.0_r2external/emma" />
      <excludeFolder url="file://$MODULE_DIR$/../10.0.0_r2external/jdiff" />
    </content>
    <orderEntry type="sourceFolder" forTests="false" />
    <orderEntry type="inheritedJdk" />
  </component>
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
如果已经把全部项目导入到Android Studio，又想删除怎么办，其实有一个简单的方法就是进入目录Project Structure -> Modules， 可快速去除某些模块, 其中红色代码Exclueded选项(即代表已删除的目录), 如下图:



# AS中关联源码，实现代码跳转
android.ipr导入AS之后，等待一段时间项目构建过程。

我们想要实现代码跳转，需要对项目进行以下几个步骤的配置：

1. 打开Project Structure
点击File菜单下的Project Structure，打开项目的配置选项。

2. 创建JDK
选择“SDKs” -> 中间栏”+“号 -> 选择新建JDK -> 配置如下（选择一个系统的，然后修改即可）:



重点：创建一个自定义名称的jdk、将Classpath和Sourcepath下的依赖都删除。

3. Android API依赖
选择目标API版本，将Java SDK修改为我们上一步创建的那个JDK，并且将Classpath和Sourcepath中的依赖清空。



配置Project
选择”Project"选项，这里很简单，只需要设置目标API即可：



配置Modules
这里很关键，我们选择“Modules”，中间栏选中项目，这里先配置API为对应版本，然后在Dependencies项目依赖选项卡中，需要删除所有能删除的选项，然后添加我们的源码目录为新的依赖项（这里我只添加了framewroks目录）。还有一点需要注意，把我们新添加的目录移动到顶部，这样就会优先从我们的源码目录查找代码了。



配置Modules 2
在Modules中，我们切换选项卡到"Sources"，然后选择我们的frameworks目录，点击"Mark as“那行中的”Source"，代表添加frameworks到源码目录。（也可以在frameworks目录上右键单击，弹框中选择“Source"选项。）

另外，如果我们不需要其他文件夹，可以使用同样类似操作，将该目录标记为”Excluded"的状态即可（添加后，会在窗口右侧展示为红色文本）。



错误处理
Mac系统上，AS启动后，提示一个错误：

Filesystem Case-Sensitivity Mismatch The project seems to be located on a case-sensitive file system. This does not match the IDE setting (controlled by property “idea.case.sensitive.fs”)

问题的原因是：在Mac和windows上默认文件系统都是不区分大小写的，而Linux和friends的文件系统则是区分大小写的。

Mac端解决办法:

显式的告诉IDE我的文件系统是区分大小写的。修改文件idea.properties，位置在IED的包内容目录（右键APP，显示包内容），在idea.properties文件内加上如下代码：

idea.case.sensitive.fs=true
