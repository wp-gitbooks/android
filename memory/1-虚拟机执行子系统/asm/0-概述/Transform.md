# 概述

## 什么是Transform

自从`1.5.0-beta1`版本开始, android gradle插件就包含了一个`Transform API`, 它允许**第三方插件在编译后的类文件转换为dex文件之前**做处理操作.
而使用`Transform API`, 我们完全可以不用去关注相关task的生成与执行流程, 它让我们可以只聚焦在如何对输入的类文件进行处理





# Transform的使用

Transform的注册和使用非常易懂, 在我们自定义的plugin内, 我们可以通过`android.registerTransform(theTransform)`或者

`android.registerTransform(theTransform, dependencies).`就可以进行注册.



```
class DemoPlugin: Plugin<Project> {
    override fun apply(target: Project) {
        val android = target.extensions.findByType(BaseExtension::class.java)
        android?.registerTransform(DemoTransform())
    }
}
```



而我们自定义的`Transform`继承于`com.android.build.api.transform.Transform`, 具体我们可以看[javaDoc](http://google.github.io/android-gradle-dsl/javadoc/1.5/com/android/build/api/transform/Transform.html#getSecondaryFileOutputs--), 以下代码是比较常见的transform处理模板

```
class DemoTransform: Transform() {
    /**
     * transform 名字
     */
    override fun getName(): String = "DemoTransform"

    /**
     * 输入文件的类型
     * 可供我们去处理的有两种类型, 分别是编译后的java代码, 以及资源文件(非res下文件, 而是assests内的资源)
     */
    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> = TransformManager.CONTENT_CLASS

    /**
     * 是否支持增量
     * 如果支持增量执行, 则变化输入内容可能包含 修改/删除/添加 文件的列表
     */
    override fun isIncremental(): Boolean = false

    /**
     * 指定作用范围
     */
    override fun getScopes(): MutableSet<in QualifiedContent.Scope> = TransformManager.SCOPE_FULL_PROJECT

    /**
     * transform的执行主函数
     */
    override fun transform(transformInvocation: TransformInvocation?) {
      transformInvocation?.inputs?.forEach {
          // 输入源为文件夹类型
          it.directoryInputs.forEach {directoryInput->
              with(directoryInput){
                  // TODO 针对文件夹进行字节码操作
                  val dest = transformInvocation.outputProvider.getContentLocation(
                      name,
                      contentTypes,
                      scopes,
                      Format.DIRECTORY
                  )
                  file.copyTo(dest)
              }
          }

          // 输入源为jar包类型
          it.jarInputs.forEach { jarInput->
              with(jarInput){
                  // TODO 针对Jar文件进行相关处理
                  val dest = transformInvocation.outputProvider.getContentLocation(
                      name,
                      contentTypes,
                      scopes,
                      Format.JAR
                  )
                  file.copyTo(dest)
              }
          }
      }
    }
}
```



每一个`Transform`都声明它的作用域, 作用对象以及具体的操作以及操作后输出的内容.

## 作用域
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210824212436.png)


通过`Transform#getScopes`方法我们可以声明自定义的transform的作用域, 指定作用域包括如下几种

| QualifiedContent.Scope |                                        |
| :--------------------- | :------------------------------------- |
| EXTERNAL_LIBRARIES     | 只包含外部库                           |
| PROJECT                | 只作用于project本身内容                |
| PROVIDED_ONLY          | 支持compileOnly的远程依赖              |
| SUB_PROJECTS           | 子模块内容                             |
| TESTED_CODE            | 当前变体测试的代码以及包括测试的依赖项 |

## 作用对象

通过`Transform#getInputTypes`我们可以声明其的作用对象, 我们可以指定的作用对象只包括两种

| QualifiedContent.ContentType |                                                             |
| :--------------------------- | :---------------------------------------------------------- |
| CLASSES                      | Java代码编译后的内容, 包括文件夹以及Jar包内的编译后的类文件 |
| RESOURCES                    | 基于资源获取到的内容                                        |

`TransformManager`整合了部分常用的Scope以及Content集合,
如果是`application`注册的transform, 通常情况下, 我们一般指定`TransformManager.SCOPE_FULL_PROJECT`;如果是`library`注册的transform, 我们只能指定`TransformManager.PROJECT_ONLY` , 我们可以在`LibraryTaskManager#createTasksForVariantScope`中看到相关的限制报错代码

```
Sets.SetView<? super Scope> difference =
        Sets.difference(transform.getScopes(), TransformManager.PROJECT_ONLY);
if (!difference.isEmpty()) {
    String scopes = difference.toString();
    globalScope
            .getAndroidBuilder()
            .getIssueReporter()
            .reportError(
                    Type.GENERIC,
                    new EvalIssueException(
                            String.format(
                                    "Transforms with scopes '%s' cannot be applied to library projects.",
                                    scopes)));
}
```



而作用对象我们主要常用到的是`TransformManager.CONTENT_CLASS`

## TransformInvocation

我们通过实现`Transform#transform`方法来处理我们的中间转换过程, 而中间相关信息都是通过`TransformInvocation`对象来传递

```
public interface TransformInvocation {

    /**
     * transform的上下文
     */
    @NonNull
    Context getContext();

    /**
     * 返回transform的输入源
     */
    @NonNull
    Collection<TransformInput> getInputs();

    /**
     * 返回引用型输入源
     */
    @NonNull Collection<TransformInput> getReferencedInputs();
    /**
     * 额外输入源
     */
    @NonNull Collection<SecondaryInput> getSecondaryInputs();

    /**
     * 输出源
     */
    @Nullable
    TransformOutputProvider getOutputProvider();


    /**
     * 是否增量
     */
    boolean isIncremental();
}
```



关于输入源, 我们可以大致分为消费型和引用型和额外的输入源

1. `消费型`就是我们需要进行transform操作的, 这类对象在处理后我们必须指定输出传给下一级,
   我们主要通过`getInputs()`获取进行消费的输入源, 而在进行变换后, 我们也必须通过设置`getInputTypes()`和`getScopes()`来指定输出源传输给下个transform.
2. 引用型输入源是指我们不进行transform操作, 但可能存在查看时候使用, 所以这类我们也不需要输出给下一级, 在通过覆写`getReferencedScopes()`指定我们的引用型输入源的作用域后, 我们可以通过`TransformInvocation#getReferencedInputs()`获取引用型输入源
3. 另外我们还可以额外定义另外的输入源供下一级使用, 正常开发中我们很少用到, 不过像是`ProGuardTransform`中, 就会指定创建mapping.txt传给下一级; 同样像是`DexMergerTransform`, 如果打开了`multiDex`功能, 则会将maindexlist.txt文件传给下一级



# Transform的原理

每个 Transform 都是一个 gradle task, 将 class 文件、本地依赖的 jar, aar 和 resource 资源统一处理。

每个 Transform 在处理完之后交给下一个 Transform。如果是用户自定义的 Transform 会插在队列的最前面

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210803212014.jpg)






## Transform的执行链

我们已经大致了解它是如何使用的, 现在看下他的原理(本篇源码基于gradle插件`3.3.2`版本)在去年[AppPlugin源码解析](https://yutiantina.github.io/2018/07/06/AppPlugin源码解析/)中, 我们粗略了解了android的`com.android.application`以及`com.android.library`两个插件都继承于`BasePlugin`, 而他们的主要执行顺序可以分为三个步骤

1. project的配置
2. extension的配置
3. task的创建

在`BaseExtension`内部维护了一个`transforms`集合对象,
`android.registerTransform(theTransform)`实际上就是将我们自定义的transform实例新增到这个列表对象中.
在`3.3.2`的源码中, 也可以这样理解. 在`BasePlugin#createAndroidTasks`中, 我们通过`VariantManager#createAndroidTasks`创建各个变体的相关编译任务, 最终通过`TaskManager#createTasksForVariantScope`(`application`插件最终实现方法在`TaskManager#createPostCompilationTasks`中, 而`library`插件最终实现方法在`LibraryTaskManager#createTasksForVariantScope`中)方法中获取`BaseExtension`中维护的`transforms`对象, 通过`TransformManager#addTransform`将对应的transform对象转换为task, 注册在`TaskFactory`中.这里关于一系列`Transform Task`的执行流程, 我们可以选择看下`application`内的相关transform流程, 由于篇幅原因, 可以自行去看相关源码, 这里的transform task流程分别是从Desugar->MergeJavaRes->自定义的transform->MergeClasses->Shrinker(包括ResourcesShrinker和DexSplitter和Proguard)->MultiDex->BundleMultiDex->Dex->ResourcesShrinker->DexSplitter, 由此调用链, 我们也可以看出在处理类文件的时候, 是不需要去考虑混淆的处理的.



## TransformManager

`TransformManager`管理了项目对应变体的所有`Transform`对象, 它的内部维护了一个`TransformStream`集合对象`streams`, 每当新增一个transform, 对应的transform会消费掉对应的流, 而后将处理后的流添加会`streams`内

```
public class TransformManager extends FilterableStreamCollection{
    private final List<TransformStream> streams = Lists.newArrayList();
}
```



我们可以看下它的核心方法`addTransform`

```
@NonNull
    public <T extends Transform> Optional<TaskProvider<TransformTask>> addTransform(
            @NonNull TaskFactory taskFactory,
            @NonNull TransformVariantScope scope,
            @NonNull T transform,
            @Nullable PreConfigAction preConfigAction,
            @Nullable TaskConfigAction<TransformTask> configAction,
            @Nullable TaskProviderCallback<TransformTask> providerCallback) {

        ...

        List<TransformStream> inputStreams = Lists.newArrayList();
        // transform task的命名规则定义
        String taskName = scope.getTaskName(getTaskNamePrefix(transform));

        // 获取引用型流
        List<TransformStream> referencedStreams = grabReferencedStreams(transform);

        // 找到输入流, 并计算通过transform的输出流
        IntermediateStream outputStream = findTransformStreams(
                transform,
                scope,
                inputStreams,
                taskName,
                scope.getGlobalScope().getBuildDir());

        // 省略代码是用来校验输入流和引用流是否为空, 理论上不可能为空, 如果为空, 则说明中间有个transform的转换处理有问题
        ...

        transforms.add(transform);

        // transform task的创建
        return Optional.of(
                taskFactory.register(
                        new TransformTask.CreationAction<>(
                                scope.getFullVariantName(),
                                taskName,
                                transform,
                                inputStreams,
                                referencedStreams,
                                outputStream,
                                recorder),
                        preConfigAction,
                        configAction,
                        providerCallback));
    }
```



在`TransformManager`中添加一个`Transform`管理, 流程可分为以下几步

1. 定义transform task名

   ```
   static String getTaskNamePrefix(@NonNull Transform transform) {
           StringBuilder sb = new StringBuilder(100);
           sb.append("transform");
   
           sb.append(
                   transform
                           .getInputTypes()
                           .stream()
                           .map(
                                   inputType ->
                                           CaseFormat.UPPER_UNDERSCORE.to(
                                                   CaseFormat.UPPER_CAMEL, inputType.name()))
                           .sorted() // Keep the order stable.
                           .collect(Collectors.joining("And")));
           sb.append("With");
           StringHelper.appendCapitalized(sb, transform.getName());
           sb.append("For");
   
           return sb.toString();
       }
   ```

从上面代码, 我们可以看到新建的transform task的命名规则可以理解为

`transform${inputType1.name}And${inputType2.name}With${transform.name}For${variantName}`, 对应的我们也可以通过已生成的transform task来验证
[![1](https://yutiantina.github.io/2019/04/24/%E6%B7%B1%E5%85%A5%E4%BA%86%E8%A7%A3TransformApi/transform1.png)](https://yutiantina.github.io/2019/04/24/深入了解TransformApi/transform1.png)



1. 通过transform内部定义的引用型输入的作用域(SCOPE)和作用类型(InputTypes), 通过求取与`streams`作用域和作用类型的交集来获取对应的流, 将其定义为我们需要的引用型流

   ```
   private List<TransformStream> grabReferencedStreams(@NonNull Transform transform) {
           Set<? super Scope> requestedScopes = transform.getReferencedScopes();
           ...
   
           List<TransformStream> streamMatches = Lists.newArrayListWithExpectedSize(streams.size());
   
           Set<ContentType> requestedTypes = transform.getInputTypes();
           for (TransformStream stream : streams) {
               Set<ContentType> availableTypes = stream.getContentTypes();
               Set<? super Scope> availableScopes = stream.getScopes();
   
               Set<ContentType> commonTypes = Sets.intersection(requestedTypes,
                       availableTypes);
               Set<? super Scope> commonScopes = Sets.intersection(requestedScopes, availableScopes);
   
               if (!commonTypes.isEmpty() && !commonScopes.isEmpty()) {
                   streamMatches.add(stream);
               }
           }
   
           return streamMatches;
       }
   ```

2. 根据transform内定义的SCOPE和INPUT_TYPE, 获取对应的消费型输入流, 在streams内移除掉这一部分消费性的输入流, 保留无法匹配SCOPE和INPUT_TYPE的流; 构建新的输出流, 并加到streams中做管理

   ```
   private IntermediateStream findTransformStreams(
               @NonNull Transform transform,
               @NonNull TransformVariantScope scope,
               @NonNull List<TransformStream> inputStreams,
               @NonNull String taskName,
               @NonNull File buildDir) {
   
           Set<? super Scope> requestedScopes = transform.getScopes();
           ...
   
           Set<ContentType> requestedTypes = transform.getInputTypes();
           // 获取消费型输入流
           // 并将streams中移除对应的消费型输入流
           consumeStreams(requestedScopes, requestedTypes, inputStreams);
   
           // 创建输出流
           Set<ContentType> outputTypes = transform.getOutputTypes();
           // 创建输出流转换的文件相关路径
           File outRootFolder =
                   FileUtils.join(
                           buildDir,
                           StringHelper.toStrings(
                                   AndroidProject.FD_INTERMEDIATES,
                                   FD_TRANSFORMS,
                                   transform.getName(),
                                   scope.getDirectorySegments()));
   
           // 输出流的创建
           IntermediateStream outputStream =
                   IntermediateStream.builder(
                                   project,
                                   transform.getName() + "-" + scope.getFullVariantName(),
                                   taskName)
                           .addContentTypes(outputTypes)
                           .addScopes(requestedScopes)
                           .setRootLocation(outRootFolder)
                           .build();
           streams.add(outputStream);
   
           return outputStream;
       }
   ```

3. 最后, 创建TransformTask, 注册到TaskManager中

   #### TransformTask

   如何触发到我们实现的`Transform#transform`方法, 就在`TransformTask`对应的TaskAction中执行

   ```
   void transform(final IncrementalTaskInputs incrementalTaskInputs)
               throws IOException, TransformException, InterruptedException {
   
           final ReferenceHolder<List<TransformInput>> consumedInputs = ReferenceHolder.empty();
           final ReferenceHolder<List<TransformInput>> referencedInputs = ReferenceHolder.empty();
           final ReferenceHolder<Boolean> isIncremental = ReferenceHolder.empty();
           final ReferenceHolder<Collection<SecondaryInput>> changedSecondaryInputs =
                   ReferenceHolder.empty();
   
           isIncremental.setValue(transform.isIncremental() && incrementalTaskInputs.isIncremental());
   
           GradleTransformExecution preExecutionInfo =
                   GradleTransformExecution.newBuilder()
                           .setType(AnalyticsUtil.getTransformType(transform.getClass()).getNumber())
                           .setIsIncremental(isIncremental.getValue())
                           .build();
   
           // 一些增量模式下的处理, 包括在增量模式下, 判断输入流(引用型和消费型)的变化
           ...
   
           GradleTransformExecution executionInfo =
                   preExecutionInfo.toBuilder().setIsIncremental(isIncremental.getValue()).build();
   
           ...
           transform.transform(
                                   new TransformInvocationBuilder(TransformTask.this)
                                           .addInputs(consumedInputs.getValue())
                                           .addReferencedInputs(referencedInputs.getValue())
                                           .addSecondaryInputs(changedSecondaryInputs.getValue())
                                           .addOutputProvider(
                                                   outputStream != null
                                                           ? outputStream.asOutput(
                                                                   isIncremental.getValue())
                                                           : null)
                                           .setIncrementalMode(isIncremental.getValue())
                                           .build());
   
                           if (outputStream != null) {
                               outputStream.save();
                           }
       }
   ```





# Transform的优化：增量与并发

到此为止，看起来Transform用起来也不难，但是，如果直接这样使用，会大大拖慢编译时间，为了解决这个问题，摸索了一段时间后，也借鉴了Android编译器中Desugar等几个Transform的实现，发现我们可以使用**增量编译**，并且上面transform方法遍历处理每个jar/class的流程，其实可以并发处理，加上一般编译流程都是在PC上，所以我们可以尽量敲诈机器的资源。

想要开启增量编译，我们需要重写Transform的这个接口，返回true。

```javascript
@Override 
public boolean isIncremental() {
    return true;
}
```

虽然开启了增量编译，但也并非每次编译过程都是支持增量的，毕竟一次clean build完全没有增量的基础，所以，我们需要检查当前编译是否是增量编译。

如果不是增量编译，则清空output目录，然后按照前面的方式，逐个class/jar处理 如果是增量编译，则要检查每个文件的Status，Status分四种，并且对这四种文件的操作也不尽相同

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210803212241.png)



NOTCHANGED: 当前文件不需处理，甚至复制操作都不用； ADDED、CHANGED: 正常处理，输出给下一个任务； REMOVED: 移除outputProvider获取路径对应的文件。 大概实现可以一起看看下面的代码

```javascript
@Override
public void transform(TransformInvocation transformInvocation){
    Collection<TransformInput> inputs = transformInvocation.getInputs();
    TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();
    boolean isIncremental = transformInvocation.isIncremental();
    //如果非增量，则清空旧的输出内容
    if(!isIncremental) {
        outputProvider.deleteAll();
    }    
    for(TransformInput input : inputs) {
        for(JarInput jarInput : input.getJarInputs()) {
            Status status = jarInput.getStatus();
            File dest = outputProvider.getContentLocation(
                    jarInput.getName(),
                    jarInput.getContentTypes(),
                    jarInput.getScopes(),
                    Format.JAR);
            if(isIncremental && !emptyRun) {
                switch(status) {
                    case NOTCHANGED:
                        break;
                    case ADDED:
                    case CHANGED:
                        transformJar(jarInput.getFile(), dest, status);
                        break;
                    case REMOVED:
                        if (dest.exists()) {
                            FileUtils.forceDelete(dest);
                        }
                        break;
                }
            } else {
                transformJar(jarInput.getFile(), dest, status);
            }
        }
        for(DirectoryInput directoryInput : input.getDirectoryInputs()) {
            File dest = outputProvider.getContentLocation(directoryInput.getName(),
                    directoryInput.getContentTypes(), directoryInput.getScopes(),
                    Format.DIRECTORY);
            FileUtils.forceMkdir(dest);
            if(isIncremental && !emptyRun) {
                String srcDirPath = directoryInput.getFile().getAbsolutePath();
                String destDirPath = dest.getAbsolutePath();
                Map<File, Status> fileStatusMap = directoryInput.getChangedFiles();
                for (Map.Entry<File, Status> changedFile : fileStatusMap.entrySet()) {
                    Status status = changedFile.getValue();
                    File inputFile = changedFile.getKey();
                    String destFilePath = inputFile.getAbsolutePath().replace(srcDirPath, destDirPath);
                    File destFile = new File(destFilePath);
                    switch (status) {
                        case NOTCHANGED:
                            break;
                        case REMOVED:
                            if(destFile.exists()) {
                                FileUtils.forceDelete(destFile);
                            }
                            break;
                        case ADDED:
                        case CHANGED:
                            FileUtils.touch(destFile);
                            transformSingleFile(inputFile, destFile, srcDirPath);
                            break;
                    }
                }
            } else {
                transformDir(directoryInput.getFile(), dest);
            }
        }
    }
}
```

这就能为我们的编译插件提供增量的特性。

实现了增量编译后，我们最好也支持**并发编译**，并发编译的实现并不复杂，只需要将上面处理单个jar/class的逻辑，并发处理，最后阻塞等待所有任务结束即可。

```javascript
private WaitableExecutor waitableExecutor = WaitableExecutor.useGlobalSharedThreadPool();
//异步并发处理jar/class
waitableExecutor.execute(() -> {
    bytecodeWeaver.weaveJar(srcJar, destJar);
    return null;
});
waitableExecutor.execute(() -> {
    bytecodeWeaver.weaveSingleClassToFile(file, outputFile, inputDirPath);
    return null;
});  
//等待所有任务结束
waitableExecutor.waitForTasksWithQuickFail(true);
```

接下来我们对编译速度做一个对比，每个实验都是5次同种条件下编译10次，去除最大大小值，取平均时间

首先，在我工作中的项目（Java/kotlin代码量行数百万级别），我们先做一次cleanbuild

```javascript
./gradlew clean assembleDebug --profile
```

给项目中添加UI耗时统计，全局每个方法（包括普通class文件和第三方jar包中的所有class）的第一行和最后一行都进行插桩，实现方式就是Transform+ASM，对比一下并发Transform和非并发Transform下，Tranform这一步的耗时

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210803212330.png)

可以发现，并发编译，基本比非并发编译速度提高了80%。效果很显著。

然后，让我们再做另一个试验，我们在项目中模拟日常修改某个class文件的一行代码，这时是符合增量编译的环境的。然后在刚才基础上还是做同样的插桩逻辑，对比增量Transform和全量Transform的差异。

```javascript
./gradlew assembleDebug --profile
```

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210803212400.png)

可以发现，增量的速度比全量的速度提升了3倍多，而且这个速度优化会随着工程的变大而更加显著。

数据表明，增量和并发对编译速度的影响是很大的。而我在查看Android gradle plugin自身的十几个Transform时，发现它们实现方式也有一些区别，有些用kotlin写，有些用java写，有些支持增量，有些不支持，而且是代码注释写了一个大大的FIXME, To support incremental build。所以，讲道理，现阶段的Android编译速度，还是有提升空间的。

上面我们介绍了Transform，以及如何高效地在编译期间处理所有字节码，那么具体怎么处理字节码呢？


# 字节码处理流水线
## Transform API

从 _Android Gradle Plugin 1.5.0-beta1_ 开始，为了简化注入自定义 _class_ 的操作，_Android_ 提供了 [Transform API](http://tools.android.com/tech-docs/new-build-system/transform-api)，允许第三方插件在 _class_ 文件被转换成 _dex_ 之前对其进行修改，在此之前，如果要实现同样的操作，只能通过 _Hook Task_ 的方式才能做到，由于 _Booster_ 仅支持 _Android Gradle Plugin 3.0.0_ 及以上版本，所以，选择了采用 [Transform API](http://tools.android.com/tech-docs/new-build-system/transform-api) 的方式，通过向 _Android Extension_ 注册 [BoosterTransform](https://github.com/didi/booster/blob/master/booster-gradle-plugin/src/main/kotlin/com/didiglobal/booster/gradle/BoosterTransform.kt) 来处理 _class_ 字节码。

## Transform Pipeline

在 _Android Gradle Plugin_ 中，通过 [Transform API](http://tools.android.com/tech-docs/new-build-system/transform-api) 注册的 `Transform` 最终会以 _Transform Pipeline_ 的形式来执行，而 _Transform Pipeline_ 是由一系列 `TransformStream` 和 `Transform` 所组成，而 `Transform` 通过 `TransformTask` 来挂载到 _Gradle_ 任务系统，并进行执行，以 _Android Gradle Plugin 3.0.0+_ 为例，_Transform Pipeline_ 在整个构建流程中的位置如下图所示：

![](data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0iVVRGLTgiIHN0YW5kYWxvbmU9Im5vIj8+PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhsaW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rIiBjb250ZW50U2NyaXB0VHlwZT0iYXBwbGljYXRpb24vZWNtYXNjcmlwdCIgY29udGVudFN0eWxlVHlwZT0idGV4dC9jc3MiIGhlaWdodD0iODE0cHgiIHByZXNlcnZlQXNwZWN0UmF0aW89Im5vbmUiIHN0eWxlPSJ3aWR0aDo3MjdweDtoZWlnaHQ6ODE0cHg7YmFja2dyb3VuZDojMDAwMDAwMDA7IiB2ZXJzaW9uPSIxLjEiIHZpZXdCb3g9IjAgMCA3MjcgODE0IiB3aWR0aD0iNzI3cHgiIHpvb21BbmRQYW49Im1hZ25pZnkiPjxkZWZzLz48Zz48IS0tTUQ1PVtmYjM3NWQyZjllY2U3NmU4MDNmODQ3ZmIxOTI2ZDNlOV0KZW50aXR5IGdlbmVyYXRlU291cmNlcy0tPjxyZWN0IGZpbGw9IiNGRkJDQkMiIGhlaWdodD0iMzYuMjk2OSIgc3R5bGU9InN0cm9rZTogIzAwMDAwMDsgc3Ryb2tlLXdpZHRoOiAxLjU7IiB3aWR0aD0iMTQwIiB4PSIxMDYiIHk9IjIxOCIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjE0IiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjEyMCIgeD0iMTE2IiB5PSIyNDAuOTk1MSI+Z2VuZXJhdGVTb3VyY2VzPC90ZXh0PjwhLS1NRDU9WzcxM2NhN2ZiNmFhMDRhNDE2NTc0NGFjYjUxZTMwNjVmXQplbnRpdHkgZ2VuZXJhdGVSZXNvdXJjZXMtLT48cmVjdCBmaWxsPSIjRkZCQ0JDIiBoZWlnaHQ9IjM2LjI5NjkiIHN0eWxlPSJzdHJva2U6ICMwMDAwMDA7IHN0cm9rZS13aWR0aDogMS41OyIgd2lkdGg9IjE1NyIgeD0iMjg5LjUiIHk9IjgiLz48dGV4dCBmaWxsPSIjMDAwMDAwIiBmb250LWZhbWlseT0ic2Fucy1zZXJpZiIgZm9udC1zaXplPSIxNCIgbGVuZ3RoQWRqdXN0PSJzcGFjaW5nQW5kR2x5cGhzIiB0ZXh0TGVuZ3RoPSIxMzciIHg9IjI5OS41IiB5PSIzMC45OTUxIj5nZW5lcmF0ZVJlc291cmNlczwvdGV4dD48IS0tTUQ1PVszM2EzMTJlYmNjNzBmOTdiZjNjNDUxNGZmZGU1MjY1NV0KZW50aXR5IGdlbmVyYXRlQXNzZXRzLS0+PHJlY3QgZmlsbD0iI0ZGQkNCQyIgaGVpZ2h0PSIzNi4yOTY5IiBzdHlsZT0ic3Ryb2tlOiAjMDAwMDAwOyBzdHJva2Utd2lkdGg6IDEuNTsiIHdpZHRoPSIxMzQiIHg9IjU4MiIgeT0iNDQ0Ii8+PHRleHQgZmlsbD0iIzAwMDAwMCIgZm9udC1mYW1pbHk9InNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMTQiIGxlbmd0aEFkanVzdD0ic3BhY2luZ0FuZEdseXBocyIgdGV4dExlbmd0aD0iMTE0IiB4PSI1OTIiIHk9IjQ2Ni45OTUxIj5nZW5lcmF0ZUFzc2V0czwvdGV4dD48IS0tTUQ1PVszMTJlYmVhMzRjMjljZTJmY2IyMDM3MDIxNzBhMmRkZV0KZW50aXR5IG1lcmdlUmVzb3VyY2VzLS0+PHJlY3QgZmlsbD0iI0ZFRkVDRSIgaGVpZ2h0PSIzNi4yOTY5IiBzdHlsZT0ic3Ryb2tlOiAjMDAwMDAwOyBzdHJva2Utd2lkdGg6IDEuNTsiIHdpZHRoPSIxMzgiIHg9IjI5OSIgeT0iMTA1Ii8+PHRleHQgZmlsbD0iIzAwMDAwMCIgZm9udC1mYW1pbHk9InNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMTQiIGxlbmd0aEFkanVzdD0ic3BhY2luZ0FuZEdseXBocyIgdGV4dExlbmd0aD0iMTE4IiB4PSIzMDkiIHk9IjEyNy45OTUxIj5tZXJnZVJlc291cmNlczwvdGV4dD48IS0tTUQ1PVtkNzU3YmQ4Njc2Y2QxYmEyZGRkMzgyZjYxYzdiMjkxOF0KZW50aXR5IG1lcmdlQXNzZXRzLS0+PHJlY3QgZmlsbD0iI0ZFRkVDRSIgaGVpZ2h0PSIzNi4yOTY5IiBzdHlsZT0ic3Ryb2tlOiAjMDAwMDAwOyBzdHJva2Utd2lkdGg6IDEuNTsiIHdpZHRoPSIxMTUiIHg9IjUxMi41IiB5PSI1NTciLz48dGV4dCBmaWxsPSIjMDAwMDAwIiBmb250LWZhbWlseT0ic2Fucy1zZXJpZiIgZm9udC1zaXplPSIxNCIgbGVuZ3RoQWRqdXN0PSJzcGFjaW5nQW5kR2x5cGhzIiB0ZXh0TGVuZ3RoPSI5NSIgeD0iNTIyLjUiIHk9IjU3OS45OTUxIj5tZXJnZUFzc2V0czwvdGV4dD48IS0tTUQ1PVsxMDdlNjFmOTk3NTExZWM3NzkwNzE2ZjE2NTY2YWQxYl0KZW50aXR5IHByb2Nlc3NSZXNvdXJjZXMtLT48cmVjdCBmaWxsPSIjRkVGRUNFIiBoZWlnaHQ9IjM2LjI5NjkiIHN0eWxlPSJzdHJva2U6ICMwMDAwMDA7IHN0cm9rZS13aWR0aDogMS41OyIgd2lkdGg9IjE0OSIgeD0iMjkzLjUiIHk9IjIxOCIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjE0IiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjEyOSIgeD0iMzAzLjUiIHk9IjI0MC45OTUxIj5wcm9jZXNzUmVzb3VyY2VzPC90ZXh0PjwhLS1NRDU9WzRmNzYzNTFhMDI2NTE0NWViNjIxMGJkMThiOTBlYTM5XQplbnRpdHkgY29tcGlsZVNvdXJjZXMtLT48cmVjdCBmaWxsPSIjRkZCQ0JDIiBoZWlnaHQ9IjM2LjI5NjkiIHN0eWxlPSJzdHJva2U6ICMwMDAwMDA7IHN0cm9rZS13aWR0aDogMS41OyIgd2lkdGg9IjEzMCIgeD0iMTExIiB5PSIzMzEiLz48dGV4dCBmaWxsPSIjMDAwMDAwIiBmb250LWZhbWlseT0ic2Fucy1zZXJpZiIgZm9udC1zaXplPSIxNCIgbGVuZ3RoQWRqdXN0PSJzcGFjaW5nQW5kR2x5cGhzIiB0ZXh0TGVuZ3RoPSIxMTAiIHg9IjEyMSIgeT0iMzUzLjk5NTEiPmNvbXBpbGVTb3VyY2VzPC90ZXh0PjwhLS1NRDU9W2EwZmM0MjE0MTIxOTIyZTY4YTg2MTVmMTIwYjgzY2NlXQplbnRpdHkgdHJhbnNmb3JtLS0+PHJlY3QgZmlsbD0iI0I3RUZDRCIgaGVpZ2h0PSIzNi4yOTY5IiBzdHlsZT0ic3Ryb2tlOiAjMDAwMDAwOyBzdHJva2Utd2lkdGg6IDEuNTsiIHdpZHRoPSI4OSIgeD0iMTMxLjUiIHk9IjQ0NCIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjE0IiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjY5IiB4PSIxNDEuNSIgeT0iNDY2Ljk5NTEiPnRyYW5zZm9ybTwvdGV4dD48IS0tTUQ1PVthOWIzM2Y2ZWNkYjczMTBhMjlkNjJhNjM5NDMxM2QyOV0KZW50aXR5IG1lcmdlRGV4LS0+PHJlY3QgZmlsbD0iI0ZFRkVDRSIgaGVpZ2h0PSIzNi4yOTY5IiBzdHlsZT0ic3Ryb2tlOiAjMDAwMDAwOyBzdHJva2Utd2lkdGg6IDEuNTsiIHdpZHRoPSI5MiIgeD0iMTMwIiB5PSI1NTciLz48dGV4dCBmaWxsPSIjMDAwMDAwIiBmb250LWZhbWlseT0ic2Fucy1zZXJpZiIgZm9udC1zaXplPSIxNCIgbGVuZ3RoQWRqdXN0PSJzcGFjaW5nQW5kR2x5cGhzIiB0ZXh0TGVuZ3RoPSI3MiIgeD0iMTQwIiB5PSI1NzkuOTk1MSI+bWVyZ2VEZXg8L3RleHQ+PCEtLU1ENT1bMThiNGMyMjBmMTdhZmEyZmExYjc5MDlmMTI1OWYyN2VdCmVudGl0eSBwYWNrYWdlLS0+PHJlY3QgZmlsbD0iI0ZFRkVDRSIgaGVpZ2h0PSIzNi4yOTY5IiBzdHlsZT0ic3Ryb2tlOiAjMDAwMDAwOyBzdHJva2Utd2lkdGg6IDEuNTsiIHdpZHRoPSI3OSIgeD0iMjUyLjUiIHk9IjY3MCIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjE0IiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjU5IiB4PSIyNjIuNSIgeT0iNjkyLjk5NTEiPnBhY2thZ2U8L3RleHQ+PCEtLU1ENT1bZTRlZGM0YzFlNzdkOGY2Y2E0ZTc4N2Y2ZmE2NGEwNmVdCmVudGl0eSBhc3NlbWJsZS0tPjxyZWN0IGZpbGw9IiNGRUZFQ0UiIGhlaWdodD0iMzYuMjk2OSIgc3R5bGU9InN0cm9rZTogIzAwMDAwMDsgc3Ryb2tlLXdpZHRoOiAxLjU7IiB3aWR0aD0iODciIHg9IjMzLjUiIHk9Ijc2NyIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjE0IiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjY3IiB4PSI0My41IiB5PSI3ODkuOTk1MSI+YXNzZW1ibGU8L3RleHQ+PCEtLU1ENT1bODNhMGIyMzI5ZTQ4MmMwYmIyZjhlZDI2Y2U2YzZlZGFdCmxpbmsgZ2VuZXJhdGVTb3VyY2VzIHRvIGNvbXBpbGVTb3VyY2VzLS0+PHBhdGggZD0iTTE3NiwyNTQuMzQ0IEMxNzYsMjczLjU2OCAxNzYsMzA0LjYzIDE3NiwzMjUuNjYxICIgZmlsbD0ibm9uZSIgaWQ9ImdlbmVyYXRlU291cmNlcy0mZ3Q7Y29tcGlsZVNvdXJjZXMiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjxwb2x5Z29uIGZpbGw9IiNBODAwMzYiIHBvaW50cz0iMTc2LDMzMC43NzgsMTgwLDMyMS43NzgsMTc2LDMyNS43NzgsMTcyLDMyMS43NzgsMTc2LDMzMC43NzgiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjEzIiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjM3IiB4PSIxODUiIHk9IjI5Ny4wNjY5Ij4qLmphdmE8L3RleHQ+PCEtLU1ENT1bZjU3OTEwZmRiMTg4MDYxYTRjOTk3M2E5OTFjMzIwYmRdCmxpbmsgZ2VuZXJhdGVSZXNvdXJjZXMgdG8gbWVyZ2VSZXNvdXJjZXMtLT48cGF0aCBkPSJNMzY4LDQ0LjQyNCBDMzY4LDU5Ljk1OSAzNjgsODIuNzgzIDM2OCw5OS42NTUgIiBmaWxsPSJub25lIiBpZD0iZ2VuZXJhdGVSZXNvdXJjZXMtJmd0O21lcmdlUmVzb3VyY2VzIiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48cG9seWdvbiBmaWxsPSIjQTgwMDM2IiBwb2ludHM9IjM2OCwxMDQuNjg2LDM3Miw5NS42ODYsMzY4LDk5LjY4NiwzNjQsOTUuNjg2LDM2OCwxMDQuNjg2IiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48IS0tTUQ1PVsxZjZjNDllMTFhN2Y5YmFhZGZhMzBhOTk4OGZhODcyNl0KbGluayBnZW5lcmF0ZUFzc2V0cyB0byBtZXJnZUFzc2V0cy0tPjxwYXRoIGQ9Ik02MzYuNjU2LDQ4MC4zNDQgQzYyMi44LDQ5OS44MTIgNjAwLjMwNCw1MzEuNDIxIDU4NS4zMzIsNTUyLjQ1OCAiIGZpbGw9Im5vbmUiIGlkPSJnZW5lcmF0ZUFzc2V0cy0mZ3Q7bWVyZ2VBc3NldHMiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjxwb2x5Z29uIGZpbGw9IiNBODAwMzYiIHBvaW50cz0iNTgyLjI1Nyw1NTYuNzc4LDU5MC43MzQ0LDU1MS43NjQ3LDU4NS4xNTYyLDU1Mi43MDQzLDU4NC4yMTY1LDU0Ny4xMjYsNTgyLjI1Nyw1NTYuNzc4IiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48IS0tTUQ1PVs2MWJkNzFlNTViYzczYTEyYTQ3NzdlOTgwMGIwYzk2NV0KbGluayBtZXJnZVJlc291cmNlcyB0byBwcm9jZXNzUmVzb3VyY2VzLS0+PHBhdGggZD0iTTM2OCwxNDEuMzQ0IEMzNjgsMTYwLjU2OCAzNjgsMTkxLjYzIDM2OCwyMTIuNjYxICIgZmlsbD0ibm9uZSIgaWQ9Im1lcmdlUmVzb3VyY2VzLSZndDtwcm9jZXNzUmVzb3VyY2VzIiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48cG9seWdvbiBmaWxsPSIjQTgwMDM2IiBwb2ludHM9IjM2OCwyMTcuNzc4LDM3MiwyMDguNzc4LDM2OCwyMTIuNzc4LDM2NCwyMDguNzc4LDM2OCwyMTcuNzc4IiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48dGV4dCBmaWxsPSIjMDAwMDAwIiBmb250LWZhbWlseT0ic2Fucy1zZXJpZiIgZm9udC1zaXplPSIxMyIgbGVuZ3RoQWRqdXN0PSJzcGFjaW5nQW5kR2x5cGhzIiB0ZXh0TGVuZ3RoPSIzMiIgeD0iMzc3IiB5PSIxODQuMDY2OSI+cmVzLyo8L3RleHQ+PCEtLU1ENT1bNTcyNmI2OGJkNzU2OWU4ZTA3MGY4OTkxOGE4ZDEwZGNdCmxpbmsgcHJvY2Vzc1Jlc291cmNlcyB0byBjb21waWxlU291cmNlcy0tPjxwYXRoIGQ9Ik0zMzguNDIxLDI1NC4xIEMzMDMuNzUsMjc0LjE0NSAyNDYuNDA4LDMwNy4yOTUgMjA5Ljk1MiwzMjguMzcxICIgZmlsbD0ibm9uZSIgaWQ9InByb2Nlc3NSZXNvdXJjZXMtJmd0O2NvbXBpbGVTb3VyY2VzIiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48cG9seWdvbiBmaWxsPSIjQTgwMDM2IiBwb2ludHM9IjIwNS42MTMsMzMwLjg4LDIxNS40MDcsMzI5Ljg0MTYsMjA5Ljk0MjUsMzI4LjM3ODksMjExLjQwNTIsMzIyLjkxNDQsMjA1LjYxMywzMzAuODgiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjEzIiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjM5IiB4PSIyOTMiIHk9IjI5Ny4wNjY5Ij5SLmphdmE8L3RleHQ+PCEtLU1ENT1bMDg2NTQ2ZjJmYThlMTE4M2UxMjhiMjQxOWRkNWNiMjhdCmxpbmsgY29tcGlsZVNvdXJjZXMgdG8gdHJhbnNmb3JtLS0+PHBhdGggZD0iTTE3NiwzNjcuMzQ0IEMxNzYsMzg2LjU2OCAxNzYsNDE3LjYzIDE3Niw0MzguNjYxICIgZmlsbD0ibm9uZSIgaWQ9ImNvbXBpbGVTb3VyY2VzLSZndDt0cmFuc2Zvcm0iIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjxwb2x5Z29uIGZpbGw9IiNBODAwMzYiIHBvaW50cz0iMTc2LDQ0My43NzgsMTgwLDQzNC43NzgsMTc2LDQzOC43NzgsMTcyLDQzNC43NzgsMTc2LDQ0My43NzgiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjEzIiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjQyIiB4PSIxODUiIHk9IjQxMC4wNjY5Ij4qLmNsYXNzPC90ZXh0PjwhLS1NRDU9W2IyODMzM2I0NzFkMTMzZjFmZDdiNTMxYmE2MTBmODRlXQpsaW5rIHRyYW5zZm9ybSB0byBtZXJnZURleC0tPjxwYXRoIGQ9Ik0xNzYsNDgwLjM0NCBDMTc2LDQ5OS41NjggMTc2LDUzMC42MyAxNzYsNTUxLjY2MSAiIGZpbGw9Im5vbmUiIGlkPSJ0cmFuc2Zvcm0tJmd0O21lcmdlRGV4IiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48cG9seWdvbiBmaWxsPSIjQTgwMDM2IiBwb2ludHM9IjE3Niw1NTYuNzc4LDE4MCw1NDcuNzc4LDE3Niw1NTEuNzc4LDE3Miw1NDcuNzc4LDE3Niw1NTYuNzc4IiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48dGV4dCBmaWxsPSIjMDAwMDAwIiBmb250LWZhbWlseT0ic2Fucy1zZXJpZiIgZm9udC1zaXplPSIxMyIgbGVuZ3RoQWRqdXN0PSJzcGFjaW5nQW5kR2x5cGhzIiB0ZXh0TGVuZ3RoPSI0MiIgeD0iMTg1IiB5PSI1MjMuMDY2OSI+Ki5jbGFzczwvdGV4dD48IS0tTUQ1PVthZmJmZDk4NWY3YTc4ZGFmOWYzY2ViMTY4NDE1MDk3ZF0KbGluayB0cmFuc2Zvcm0gdG8gcGFja2FnZS0tPjxwYXRoIGQ9Ik0yMDQuODA4LDQ4MC4xODggQzIxNS45MDUsNDg4LjA1NyAyMjcuODU2LDQ5OC4yNTcgMjM2LDUxMCBDMjY5Ljk2Myw1NTguOTcyIDI4NC4wNzksNjI5LjU0MSAyODkuMyw2NjQuOTYgIiBmaWxsPSJub25lIiBpZD0idHJhbnNmb3JtLSZndDtwYWNrYWdlIiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48cG9seWdvbiBmaWxsPSIjQTgwMDM2IiBwb2ludHM9IjI5MC4wMDcsNjY5LjkyNCwyOTIuNjk2Nyw2NjAuNDQ5NSwyODkuMzAxMyw2NjQuOTc0MSwyODQuNzc2OCw2NjEuNTc4NywyOTAuMDA3LDY2OS45MjQiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjEzIiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjI1IiB4PSIyODMiIHk9IjU3OS41NjY5Ij4qLnNvPC90ZXh0PjwhLS1NRDU9Wzc0MWJlODJjZWZkM2JhZGExODZiMGE0YWY5Y2VlYTVjXQpsaW5rIG1lcmdlQXNzZXRzIHRvIHBhY2thZ2UtLT48cGF0aCBkPSJNNTI3LjE3Miw1OTMuMSBDNDc1LjQyNiw2MTMuNzYyIDM4OC44MDMsNjQ4LjM0OSAzMzYuMzc0LDY2OS4yODIgIiBmaWxsPSJub25lIiBpZD0ibWVyZ2VBc3NldHMtJmd0O3BhY2thZ2UiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjxwb2x5Z29uIGZpbGw9IiNBODAwMzYiIHBvaW50cz0iMzMxLjcyMiw2NzEuMTQsMzQxLjU2MzYsNjcxLjUxNzcsMzM2LjM2NTYsNjY5LjI4NiwzMzguNTk3Miw2NjQuMDg4LDMzMS43MjIsNjcxLjE0IiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48dGV4dCBmaWxsPSIjMDAwMDAwIiBmb250LWZhbWlseT0ic2Fucy1zZXJpZiIgZm9udC1zaXplPSIxMyIgbGVuZ3RoQWRqdXN0PSJzcGFjaW5nQW5kR2x5cGhzIiB0ZXh0TGVuZ3RoPSI1NCIgeD0iNDU3IiB5PSI2MzYuMDY2OSI+YXNzZXRzLyo8L3RleHQ+PCEtLU1ENT1bZGVmYmVhYjI3NzY3YTdhZjQzNTlhNzYyMTFhNDVkNzZdCmxpbmsgcHJvY2Vzc1Jlc291cmNlcyB0byBwYWNrYWdlLS0+PHBhdGggZD0iTTM3MS4wMjgsMjU0LjI0MiBDMzc0LjU4MywyNzYuMDE0IDM4MCwzMTQuNjYxIDM4MCwzNDggQzM4MCwzNDggMzgwLDM0OCAzODAsNTc2IEMzODAsNjE0LjI4MyAzNDcuNDYsNjQ3LjA1MiAzMjEuOTU0LDY2Ni44NTcgIiBmaWxsPSJub25lIiBpZD0icHJvY2Vzc1Jlc291cmNlcy0mZ3Q7cGFja2FnZSIgc3R5bGU9InN0cm9rZTogI0E4MDAzNjsgc3Ryb2tlLXdpZHRoOiAxLjA7Ii8+PHBvbHlnb24gZmlsbD0iI0E4MDAzNiIgcG9pbnRzPSIzMTcuOTI3LDY2OS45MjMsMzI3LjUxMDYsNjY3LjY1MjgsMzIxLjkwNDksNjY2Ljg5MzgsMzIyLjY2NCw2NjEuMjg4MSwzMTcuOTI3LDY2OS45MjMiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjEzIiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjE1NyIgeD0iMzg5IiB5PSI0NjYuNTY2OSI+cmVzb3VyY2VzLXt2YXJpYW50fS5hcF88L3RleHQ+PCEtLU1ENT1bY2M4ODRmODFmYmI3YWQ1NGRmYzMyMDg3NzMyNjI5YWNdCmxpbmsgbWVyZ2VEZXggdG8gcGFja2FnZS0tPjxwYXRoIGQ9Ik0xNjIuNTcsNTkzLjA0MSBDMTUzLjUzMiw2MDYuNzI2IDE0NC45NDIsNjI1LjkxNiAxNTUsNjQwIEMxNzUuODA3LDY2OS4xMzYgMjE1Ljg0OSw2ODAuNDAxIDI0Ny4yOTIsNjg0LjY2MSAiIGZpbGw9Im5vbmUiIGlkPSJtZXJnZURleC0mZ3Q7cGFja2FnZSIgc3R5bGU9InN0cm9rZTogI0E4MDAzNjsgc3Ryb2tlLXdpZHRoOiAxLjA7Ii8+PHBvbHlnb24gZmlsbD0iI0E4MDAzNiIgcG9pbnRzPSIyNTIuMjYxLDY4NS4yODYsMjQzLjgzMjQsNjgwLjE5MSwyNDcuMzAwMyw2ODQuNjYwMiwyNDIuODMxMSw2ODguMTI4MSwyNTIuMjYxLDY4NS4yODYiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjx0ZXh0IGZpbGw9IiMwMDAwMDAiIGZvbnQtZmFtaWx5PSJzYW5zLXNlcmlmIiBmb250LXNpemU9IjEzIiBsZW5ndGhBZGp1c3Q9InNwYWNpbmdBbmRHbHlwaHMiIHRleHRMZW5ndGg9IjEwMSIgeD0iMTY0IiB5PSI2MzYuMDY2OSI+Y2xhc3Nlc3tOfS5kZXg8L3RleHQ+PCEtLU1ENT1bNGFjMjY1MDZlZDlhYTQ5YzJlNGM5YmY5M2M1ODhkYjFdCmxpbmsgY29tcGlsZVNvdXJjZXMgdG8gYXNzZW1ibGUtLT48cGF0aCBkPSJNMTEwLjc2LDM2NC43NDQgQzYyLjgxODgsMzc5Ljg4NCA2LDQwOC44MzMgNiw0NjEgQzYsNDYxIDYsNDYxIDYsNjg5IEM2LDcxOS4zMDY0IDI5LjQ3MDQsNzQ2LjE2NzQgNDkuNDIyNSw3NjMuNTI2NiAiIGZpbGw9Im5vbmUiIGlkPSJjb21waWxlU291cmNlcy0mZ3Q7YXNzZW1ibGUiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjxwb2x5Z29uIGZpbGw9IiNBODAwMzYiIHBvaW50cz0iNTMuMzU4LDc2Ni44Njk5LDQ5LjA4ODgsNzU3Ljk5NDQsNDkuNTQ3NSw3NjMuNjMyNiw0My45MDkyLDc2NC4wOTEzLDUzLjM1OCw3NjYuODY5OSIgc3R5bGU9InN0cm9rZTogI0E4MDAzNjsgc3Ryb2tlLXdpZHRoOiAxLjA7Ii8+PCEtLU1ENT1bMWYxNWI1OGQ2YzMyMjFiMmY2ZTFhNmM3YzhiMDM1MDldCmxpbmsgdHJhbnNmb3JtIHRvIGFzc2VtYmxlLS0+PHBhdGggZD0iTTE2MC45MTUsNDgwLjAwOSBDMTQ1Ljk2Miw0OTcuODcyIDEyMy42NTMsNTI3LjQyNyAxMTIsNTU3IEM4My45NjEzLDYyOC4xNTUgNzguMjkwMyw3MjAuMTA3NyA3Ny4yMDkzLDc2MS43NTQ3ICIgZmlsbD0ibm9uZSIgaWQ9InRyYW5zZm9ybS0mZ3Q7YXNzZW1ibGUiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyBzdHJva2UtZGFzaGFycmF5OiA3LjAsNy4wOyIvPjxwb2x5Z29uIGZpbGw9IiNBODAwMzYiIHBvaW50cz0iNzcuMDk0NCw3NjYuODE2OCw4MS4yOTc1LDc1Ny45MDk5LDc3LjIwNzgsNzYxLjgxODEsNzMuMjk5Niw3NTcuNzI4NCw3Ny4wOTQ0LDc2Ni44MTY4IiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48IS0tTUQ1PVs3NTk1NzQyYmE1MDlmYTZiMjNhYjBiY2YxN2E2YTNhNF0KbGluayBwYWNrYWdlIHRvIGFzc2VtYmxlLS0+PHBhdGggZD0iTTI1My4wNjUsNzA2LjIwMzggQzIxNS42MjIsNzIyLjc0ODQgMTU5LjEzMiw3NDcuNzA5MSAxMjAuMjkyLDc2NC44NzEgIiBmaWxsPSJub25lIiBpZD0icGFja2FnZS0mZ3Q7YXNzZW1ibGUiIHN0eWxlPSJzdHJva2U6ICNBODAwMzY7IHN0cm9rZS13aWR0aDogMS4wOyIvPjxwb2x5Z29uIGZpbGw9IiNBODAwMzYiIHBvaW50cz0iMTE1LjY0NSw3NjYuOTI0MiwxMjUuNDkzOCw3NjYuOTQ0NCwxMjAuMjE4Miw3NjQuOTAyOSwxMjIuMjU5Nyw3NTkuNjI3MiwxMTUuNjQ1LDc2Ni45MjQyIiBzdHlsZT0ic3Ryb2tlOiAjQTgwMDM2OyBzdHJva2Utd2lkdGg6IDEuMDsiLz48IS0tTUQ1PVtkOGUzYmY1NDE3OGI4ODJjODZlYWRhMGMyYzdhYTBjMl0KQHN0YXJ0dW1sDQpza2lucGFyYW0gcmVjdGFuZ2xlPDxiZWhhdmlvcj4+IHsNCiAgICByb3VuZENvcm5lciAyNQ0KfQ0Kc2tpbnBhcmFtIGJhY2tncm91bmRjb2xvciB0cmFuc3BhcmVudA0Kc2tpbnBhcmFtIHNoYWRvd2luZyBmYWxzZQ0KDQpyZWN0YW5nbGUgZ2VuZXJhdGVTb3VyY2VzICAgICNmZmJjYmMNCnJlY3RhbmdsZSBnZW5lcmF0ZVJlc291cmNlcyAgI2ZmYmNiYw0KcmVjdGFuZ2xlIGdlbmVyYXRlQXNzZXRzICAgICAjZmZiY2JjDQpyZWN0YW5nbGUgbWVyZ2VSZXNvdXJjZXMNCnJlY3RhbmdsZSBtZXJnZUFzc2V0cw0KcmVjdGFuZ2xlIHByb2Nlc3NSZXNvdXJjZXMNCnJlY3RhbmdsZSBjb21waWxlU291cmNlcyAgICAgI2ZmYmNiYw0KcmVjdGFuZ2xlIHRyYW5zZm9ybSAgICAgICAgICAjYjdlZmNkDQpyZWN0YW5nbGUgbWVyZ2VEZXgNCnJlY3RhbmdsZSBwYWNrYWdlDQpyZWN0YW5nbGUgYXNzZW1ibGUNCg0KZ2VuZXJhdGVTb3VyY2VzICAgLWQtPiBjb21waWxlU291cmNlcyA6ICIgICouamF2YSINCmdlbmVyYXRlUmVzb3VyY2VzIC1kLT4gbWVyZ2VSZXNvdXJjZXMNCmdlbmVyYXRlQXNzZXRzICAgIC1kLT4gbWVyZ2VBc3NldHMNCm1lcmdlUmVzb3VyY2VzICAgIC1kLT4gcHJvY2Vzc1Jlc291cmNlcyA6ICIgIHJlcy8qIg0KcHJvY2Vzc1Jlc291cmNlcyAgLWQtPiBjb21waWxlU291cmNlcyA6ICIgIFIuamF2YSINCmNvbXBpbGVTb3VyY2VzICAgIC1kLT4gdHJhbnNmb3JtIDogIiAgKi5jbGFzcyINCnRyYW5zZm9ybSAgICAgICAgIC1kLT4gbWVyZ2VEZXggOiAiICAqLmNsYXNzIg0KdHJhbnNmb3JtICAgICAgICAgLWQtPiBwYWNrYWdlIDogIiAgKi5zbyINCm1lcmdlQXNzZXRzICAgICAgIC1kLT4gcGFja2FnZSA6ICIgIGFzc2V0cy8qIg0KcHJvY2Vzc1Jlc291cmNlcyAgLWQtPiBwYWNrYWdlIDogIiAgcmVzb3VyY2VzLXt2YXJpYW50fS5hcF8iDQptZXJnZURleCAgICAgICAgICAtZC0+IHBhY2thZ2UgOiAiICBjbGFzc2Vze059LmRleCINCmNvbXBpbGVTb3VyY2VzICAgIC1kLT4gYXNzZW1ibGUNCnRyYW5zZm9ybSAgICAgICAgIC4uLj4gYXNzZW1ibGUNCnBhY2thZ2UgICAgICAgICAgIC1kLT4gYXNzZW1ibGUNCkBlbmR1bWwNCgpQbGFudFVNTCB2ZXJzaW9uIDEuMjAyMC4wOWJldGExNShVbmtub3duIGNvbXBpbGUgdGltZSkKKEdQTCBzb3VyY2UgZGlzdHJpYnV0aW9uKQpKYXZhIFJ1bnRpbWU6IEphdmEoVE0pIFNFIFJ1bnRpbWUgRW52aXJvbm1lbnQKSlZNOiBKYXZhIEhvdFNwb3QoVE0pIDY0LUJpdCBTZXJ2ZXIgVk0KSmF2YSBWZXJzaW9uOiAxNC4wLjErNwpPcGVyYXRpbmcgU3lzdGVtOiBMaW51eApEZWZhdWx0IEVuY29kaW5nOiBVVEYtOApMYW5ndWFnZTogZW4KQ291bnRyeTogVVMKLS0+PC9nPjwvc3ZnPg==)

### transform Stream

每个 _Transform Stream_ 都有特定的类型（Content Types）和作用域（Scopes）。

#### Content Types

内容类型

描述

`QualifiedContent.DefaultContentType.CLASSES`

_class_ 或者 _JAR_ 文件

`QualifiedContent.DefaultContentType.RESOURCES`

Java 资源

`ExtendedContentType.DEX`

_dex_ 文件

`ExtendedContentType.NATIVE_LIBS`

_so_ 动态库

`ExtendedContentType.CLASSES_ENHANCED`

_Instant Run_ 更新的 _class_

`ExtendedContentType.DATA_BINDING`

_data binding_ 产物

`ExtendedContentType.DEX_ARCHIVE`

_dex_ 存档文件，每个 _class_ 对应一个单独的 _dex_ 文件

由于 _Booster_ 只关注于字节码的处理，所以 [BoosterTransform](https://github.com/didi/booster/blob/master/booster-gradle-plugin/src/main/kotlin/com/didiglobal/booster/gradle/BoosterTransform.kt) 所接收的输入内容类型为 `QualifiedContent.DefaultContentType.CLASSES`。

#### Scopes

作用域

描述

`QualifiedContent.Scope.PROJECT`

仅作用于当前工程

`QualifiedContent.Scope.SUB_PROJECTS`

作用于所有子工程

`QualifiedContent.Scope.EXTERNAL_LIBRARIES`

作用于外部依赖库

`QualifiedContent.Scope.TESTED_CODE`

作用于当前 _variant_ 的测试代码

`QualifiedContent.Scope.PROVIDED_ONLY`

仅作用于 _provided_ 的依赖

`QualifiedContent.Scope.PROJECT_LOCAL_DEPS`

作用于工程的本地依赖包

`QualifiedContent.Scope.SUB_PROJECTS_LOCAL_DEPS`

作用于所有子工程的本地依赖包

`InternalScope.MAIN_SPLIT`

作用于 _Instant Run_ 模式下主包

`InternalScope.LOCAL_DEPS`

作用于本地依赖库

`InternalScope.FEATURES`

作用于 _Dynamic Feature_ 工程

`TransformManager` 中定义了一些常用的 _Scopes_：

作用域

描述

`TransformManager.SCOPE_FULL_PROJECT`

作用于整个工程，包括当前工程、所有子工程及外部依赖库

`TransformManager.PROJECT_ONLY`

仅作用于当前工程，不包括子工程或外部依赖库

`TransformManager.SCOPE_FULL_WITH_FEATURES`

作用于整个工程以及 _Dynamic Feature_ 工程

`TransformManager.SCOPE_FULL_LIBRARY_WITH_LOCAL_JARS`

作用于当前工程以及本地依赖库

对于 _APP_ 工程而言， [BoosterTransform](https://github.com/didi/booster/blob/master/booster-gradle-plugin/src/main/kotlin/com/didiglobal/booster/gradle/BoosterTransform.kt) 的作用域为 `TransformManager.SCOPE_FULL_PROJECT`，不仅要处理当前工程自身的字节码，还要处理其依赖的本地库或者外部库等；对于 _Library_ 工程而言，其作用域为 `TransformManager.PROJECT_ONLY`；对于 _dynamic-feature_ 工程而言，则为 `TransformManager.SCOPE_FULL_WITH_FEATURES`

#### Original Stream & Intermediate Stream

根据 _Transform Stream_ 的起源，又将其分为原始流（Original Stream）和中间流（Intermediate Stream），顾名思义，原始流是由 _Android Gradle Plugin_ 创建，然后给其它的 `Transform` 来消费其中的内容，而自定义的 `Transform` 消费后产生的 _Transform Stream_ 则为中间流（非原始的）。

当我们通过自定义 `Transform` 来处理构建中间产物，可以从两个维度来关注：

1.  _Content Types_：即 `Transform` 要处理的内容是什么类型，可以是多种类型；
2.  _Scopes_：即 `Transform` 要处理哪里的内容，可以是多个作用域；

当通过 _Transform API_ 注册了自定义的 `Transform` 后，_Transform Manager_ 就会根据 `Transform` 关注的 _Content Types_ 和 _Scopes_ 从现有的 _Transform Stream_ 列表中选择对类型和作用域的流作为该 `Transform` 的输入，而该 `Transform` 的输出（_Intermediate Stream_）则作为下一个 `Transform` 的输入，每个 `Transform` 通过 _Transform Stream_ 连接起来，这样就形成了管道，而 [BoosterTransform](https://github.com/didi/booster/blob/master/booster-gradle-plugin/src/main/kotlin/com/didiglobal/booster/gradle/BoosterTransform.kt) 则是 _Transform Pipeline_ 中的一个节点，如下图所示：

![Transform Pipeline](data:image/png;base64,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IIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAggggAACCCCAAAIIIIAAAlUi8H8lGSjeQNdDRQAAAABJRU5ErkJggg==)

## Booster Transformer Pipeline

为了简化对字节码的操作以及提升模块的可复用性，_Booster_ 内部设计了一套类似于 _Android Transform Pipeline_ 的 _Transformer Pipeline_ 框架来实现字节码流式操作， `Transformer` 作为字节码的处理单元，开发者可以根据个人喜好来选择 `JavassistTransformer`、`AsmTransformer` 或者基于其它字节码操作框架自定义 `Transformer`，在 `JavassistTransformer` 和 `AsmTransformer` 内部，还有一层 _Class Transformer Pipeline_，每个 _ClassTransformer_ 作为 _class_ 的处理单元，_Transformer Pipeline_ 的结构如下图所示：

![Booster Transformer Pipeline](data:image/png;base64,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)

### 并行 Transformer Pipeline

为了最大限度的提升构建速度，_Booster_ 为每一个输入（`JarInput` 或 `DirectoryInput`）创建了一条独立的 _Transformer Pipeline_，所有 _Transformer Pipeline_ 共享 `Transformer`。

### 调整 Transformer 在 Pipeline 中的顺序

默认情况下，_Booster_ 是不保证 `Transformer` 的执行顺序的，这样的好处是减少 `Transformer` 之间的依赖，降低 _Transform Pipeline_ 的复杂度，但总有一些特殊情况存在，为了满足 `Transformer` 的顺序可调整，_Booster_ 提供了通过在 `Transformer` 上标注 [@Priority](https://github.com/didi/booster/blob/master/booster-annotations/src/main/kotlin/com/didiglobal/booster/annotations/Priority.kt) 的方式为 `Transformer` 指定优先级（值越小优先级越高），例如：

```
@Priority(Int.MAX_VALUE)
@AutoService(ClassTransformer::class)
class HighestPriorityTransformer : ClassTransformer {

    override fun transform(context: TransformContext, klass: ClassNode): ClassNode {
        // do something...
        return super.transform(context, klass)
    }

}
```

> `@Priority` 同样适用于调整 `VariantProcessor` 的执行顺序

## 上帝视角

至此，我们已经了解了 _Transform Pipeline_ 的内部实现细节，如何自定义一个 `Transform` 只对当前 _Transform Pipeline_ 中的内容进行观察而不修改呢？这个问题就留给本书的读者吧。



# 参考

https://cloud.tencent.com/developer/article/1378925

https://booster.johnsonlee.io/architecture/transformer-pipeline.html#transform-api


# 面试题
## transform输入参数

## android中有哪些常见的transform？

## 