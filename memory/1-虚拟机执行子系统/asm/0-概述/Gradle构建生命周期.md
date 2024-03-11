# 线索  [[Gradle-面试题]]
![image-20210723113748977](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210723113749.png)



# 两个重要的概念

## 项目

实际上，一个项目是什么取决于你要用 Gradle 做什么？项目通常代表的是构建内容。 例如在 Android 中，一个 module 就是一个项目；

- 项目是注册在 settings.gradle 中的
- 通常一个项目有一个 build.gradle

Gradle 构建就是由一个或多个项目组成的。

## 任务

任务 顾名思义就是一个在构建阶段被执行的操作。它是 Gradle 构建的原子工作单位。例如 编译 Java 源代码；

任务是定义在项目的构建脚本中，并且可以彼此依赖。

一个项目就是由一个个任务组成的。

# 生命周期  [[Gradle生命周期详解]]
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210801211550.png)


>  每一个 Gradle 构建都会按照相同的顺序经历三个不同的阶段： 

## 初始化

Gradle 支持单项目构建和多项目构建。 在这个阶段 Gradle 会确认哪些项目将会参与构建。Gradle 会通过 settings.gradle 确定是多项目还是单项目构建。 Gradle 会为每个项目创建 Project 实例。

## 配置

在这个阶段执行在初始化阶段中确定的每一个项目的配置脚本，但是并不会执行其中的任务，只会评估任务的依赖性，根据其依赖性创建任务的有向无环图。

Gradle引入了一个称为随需求变配置的特性，该特性使它能够在构建过程中只配置相关和必要的项目。这在大型多项目构建中非常有用，因为它可以大大减少构建时间。

## 执行

在这个阶段，Gradle 会识别在配置阶段创建的任务的有向无环图。并按照他们的依赖顺序开始执行。 所有的构建工作都是在这个阶段执行的。如编译源码，生成 .class 文件，复制文件等。

# setting.gradle

这个文件是由 Gradle 约定命名的，默认名为 settings.gradle ，在初始化阶段被执行。

对于多项目构建，必须在这里声明要参与构建的所有项目。对于单项目构建就是可选的了，可有可无。

Gradle 是如何寻找 settings.gradle 的？

1. 在当前目录寻找
2. 没有找到的话就去父目录寻找
3. 仍然没有找到就是是单项目构建了
4. 如果找到了就是确定其中的项目，如果当前执行的项目在 settings.gradle 有定义就执行多项目构建，否则就执行单项目构建。

一个脚本的属性访问和方法调用是委托给 Project 类的实例的，类似的 settings.gradle 的属性访问和方法调用是委托给 Settings 类的实例对象的。

# 单项目构建

对于单项目构建，在初始化后的工作流程很简单，构建脚本针对初始化阶段创建的项目对象执行。查找在命令行传入的任务名称相同的任务。 如果任务存在则作为一个单独的构建按照命令行传递的顺序执行。

# 多项目构建

多项目构建是在 Gradle 的单个执行过程中构建多个项目的构建。必须把参与构建的项目声明在 settings.gradle 里

## 项目位置

可以把多项目构建看作一个单根的树。每一个项目都是树上的一个节点。一个项目有一个路径表示在树中的位置。 通常情况下项目的路径和在文件系统中的位置是一致的，当然了这个路径也是可以配置的。 项目树是 settings.gradle 生成的，默认情况下 settings.gradle 的位置就是根项目的位置。但是你可以在 settings.gradle 文件中更改。

## 构建项目树

在 settings.gradle 设置文件中你可以使用一些列的方法配置构建项目树。分层和平面物理布局都支持。

### 分层布局

Groovy

```javascript
include 'project1', 'project2:child', 'project3:child1'
```

Kotlin

```javascript
include("project1", "project2:child", "project3:child1")
```

include 方法使用项目路径作为参数，假定项目路径与相对物理文件系统路径相等。 例如 “project2:child” 默认对应的是相对于根目录的 “project2/child”。 这也意味着包含路径 “services:hotels:api” 将创建3个项目:

- “services”
- “services:hotels”
- “services:hotels:api”

更详细的说明可以 [DSL文档](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html#org.gradle.api.initialization.Settings:include(java.lang.String[]))

### 平面布局

Groovy

```javascript
includeFlat 'project3', 'project4'
```

Kotlin

```javascript
includeFlat("project3", "project4")
```

includeFlat 也是目录名字作为参数。这些目录要和根项目目录同级。 这些目录的位置在项目树中是根项目的子项目。

## 更改项目树的元素

在设置文件中创建的多项目树由所谓的项目描述符组成。这些项目符号可以随时更改。 可以通过下面这种方式访问描述符

>  查找项目树的元素 

Groovy

```javascript
println rootProject.name
println project(':projectA').name
```

Kotlin

```javascript
println(rootProject.name)
println(project(":projectA").name)
```

使用这个描述符你可以一个项目的名字，项目目录和构建文件

>  更改项目树元素 

Groovy

```javascript
rootProject.name = 'main'
project(':projectA').projectDir = new File(settingsDir, '../my-project-a')
project(':projectA').buildFileName = 'projectA.gradle'
```

Kotlin

```javascript
rootProject.name = "main"
project(":projectA").projectDir = File(settingsDir, "../my-project-a")
project(":projectA").buildFileName = "projectA.gradle"
```

更详细的信息可以查看 [ProjectDescriptor](https://docs.gradle.org/current/javadoc/org/gradle/api/initialization/ProjectDescriptor.html) 类的 API 文档。

# 接收生命周期事件

构建脚本可以接收生命周期构建进度的通知。

接收这些通知一般是两种形式

- 实现详细的监听接口
- 在发送通知时提供一个闭包来执行

## 项目评估事件

可以在项目评估后马上接到事件通知 使用的是 Project.afterEvaluate 方法，传入一个闭包，Gradle会将评估的项目和状态传递进闭包里。 Kotlin

```javascript
afterEvaluate {
      println("${project.getName()} 评估结果：${state.getExecuted()}")
   }
```

Groovy

```javascript
afterEvaluate{ project,state->
    println "$project 评估成功否：${state.failure==null}"
}
```

如果是在多项目构建里，可以在 allprojects 的闭包里使用，这样每个项目的评估事件就都接受到了 Groovy

```javascript
allprojects{
    afterEvaluate{ project,state->
    println "$project 评估成功否：${state.failure==null}"
    }
}
```

评估前的事件通知使用 Project.beforeEvaluate 照样是传入一个闭包，Gradle会将要评估的项目传递进闭包里

Groovy

```javascript
allprojects{
    afterEvaluate{ project,state->
        println "$project 评估成功否：${state.failure==null}"
    }

   beforeEvaluate { project ->
       println "开始评估 $project"
   }

}
```

这里列出了使用的 api文档。

- [Project](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html)
- [Project.allprojects](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html#allprojects-groovy.lang.Closure-)
- [Project.afterEvaluate](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html#afterEvaluate-groovy.lang.Closure-)
- [Project.beforeEvaluate](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html#beforeEvaluate-groovy.lang.Closure-)
- [ProjectState](https://docs.gradle.org/current/javadoc/org/gradle/api/ProjectState.html)

## 任务

### 任务被添加到项目

Groovy

```javascript
tasks.whenTaskAdded { task ->
   println "$task 被添加到项目了。"
}
```

Kotlin

```javascript
tasks.whenTaskAdded {
    extra["srcDir"] = "src/main/java"
}

val a by tasks.registering

println("source dir is ${a.get().extra["srcDir"]}")
```

### 有向无环图填充完毕

使用的是 TaskExecutionGraph.whenReady 方法

Groovy

```javascript
gradle.taskGraph.whenReady{ graph->
   println "任务图准备好了：\n"
   graph.allTasks.each {
       print "$it , "
   }
}
```

- [TaskExecutionGraph](https://docs.gradle.org/current/javadoc/org/gradle/api/execution/TaskExecutionGraph.html#whenReady-groovy.lang.Closure-)

### 任务执行

Groovy

```javascript
task ok

task broken(dependsOn: ok) {
    doLast {
        throw new RuntimeException('broken')
    }
}

gradle.taskGraph.beforeTask { Task task ->
    println "executing $task ..."
}

gradle.taskGraph.afterTask { Task task, TaskState state ->
    if (state.failure) {
        println "FAILED"
    }
    else {
        println "done"
    }
}
```

# 参考

https://cloud.tencent.com/developer/article/1552131





- [TaskExecutionGraph](https://docs.gradle.org/current/javadoc/org/gradle/api/execution/TaskExecutionGraph.html#whenReady-groovy.lang.Closure-)

这里留一个[Gradle API 的查询地址](https://docs.gradle.org/current/javadoc/index.html)

文档参考

- https://docs.gradle.org/current/userguide/build_lifecycle.html
- https://proandroiddev.com/understanding-gradle-the-build-lifecycle-5118c1da613f

