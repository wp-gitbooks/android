# 概述

在软件业，AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

目前AOP技术已经在移动APP开发过程中广泛应用，在京东内部已经落地实践了**性能监控、隐私方法/字段合规检测、线程使用数优化**等场景，AOP技术的一个特性就是业务的无侵入性，相比较传统的开发模式极大减少了接入工作量，同时具备了覆盖面广的优势，可以对宿主APP中的任意代码进行Hook替换，可以让开发者从‘上帝视角’去操作。

本文将对**安全扫描、线程优化** **两个主要场景展开描述**，从而介绍AOP技术在移动端多场景下的优势。


## AOP技术介绍

常见的AOP工具按照生效时机区分主要分为两大类：预编译期及运行期，以下列举出市面上常用的AOP工具及对应开源框架:

### APT工具

代表开源框架：ButterKnife、Dagger2、DBFlow、AndroidAnnotation 注解处理器 Java5 中叫APT(Annotation Processing Tool)，在Java6开始，规范化为 Pluggable Annotation Processing。Apt应该是这其中我们最常见到的了，难度也最低。定义编译期的注解，再通过继承Proccesor实现代码生成逻辑，实现了编译期生成代码的逻辑。



### AspectJ工具

代表开源框架：Hugo(Jake Wharton)，360 Argus等；



### Javassist/ASM工具

代表开源框架：热修复框架HotFix 、Savior（InstantRun），听云APM，Google FireBase，百度云等；



### Dexposed-XPosed工具

Dexposed是基于久负盛名的开源Xposed框架实现的一个Android平台上功能强大的无侵入式运行时AOP框架。Dexposed的AOP实现是完全非侵入式的，没有使用任何注解处理器，编织器或者字节码重写器。基于动态类加载技术，运行中的app可以加载一小段经过编译的Java AOP代码，在不需要重启app的前提下实现修改目标app的行为。



# AOP开发实践

移动端AOP技术的使用主要用于非业务侵入式的开发场景中，例如日志埋点、性能监控、数据校验、持久化、动态权限控制、甚至是代码调试等等，接下来对京东内部使用AOP解决问题的两个场景进行介绍。



## 场景一 : 隐私方法/字段合规检测




### 背景介绍

国内外用户隐私数据相关法律法规日趋完善、行政管制也趋于常态化、用户隐私数据安全意识也在逐步提升，移动应用上线后出现隐私数据安全合规问题的风险也越发不可控。在此场景下，集团内部保护用户隐私权利的角度出发，针对移动APP展开了用户隐私数据使用合规化整改。



### 问题描述

在移动端应用开发过程中常规的业务或者独立SDK有采集设备或者个人信息的场景，对于涉及隐私权限的接口（例如接打电话、短信、定位等）Android系统提供了Runtime期间用户授权的机制，用户选择性开启相关权限达到安全访问的目的，此外还有个人常用设备信息（进程列表、系统版本号、厂商信息等）系统未提供权限授权机制，这类信息为合规化整治的重点。针对京东APP这样的大型APP来讲，涉及到大量的业务团队以及第三方SDK的改造，需要快速地定位问题并推动问题模块整改，我们选取了AOP插件的方式来进行问题检测及定位，从而避免了各业务模块自查的耗时操作。



### 技术实施

在明确了需要进行切片的代码点后，结合各个点的访问方式的区别（分为接口调用、类成员访问），我们选取ASM框架编写Gradle插件在编译期hook指定的代码以及结合Xposed运行时查找指定代码调用的方式来进行相关隐私接口调用代码的监控工作，理由如下：

\1. 通过ASM框架实现编译期字节码插桩可以实现hook java类中所有类型的调用场景，例如方法调用，静态常量调用，成员变量，对象构造器等，但也有一定的局限性，首先上手难度较高，开发工作量较大且需要对java字节码结构有一定的了解，其次就是只能处理编译期输入的class文件（应用的工程代码)，对于jni层以及Android framework层的代码间接调用无法进行hook，这样可能对AOP结果的可靠性有一定的挑战。

\2. 基于ASM字节码插桩的局限性，随后采用了基于Xposed的运行期的AOP框架epic工具来作为补偿，Xposed框架进行代码hook原理为通过在zygote进程fork子进程当做应用的运行进程的过程中加载了一个额外的jar包，以此来实现对系统底层代码的hook操作，但是当前该框架只是提供了对方法的hook以及回调，没有对其他类型（成员变量调用等）提供相关的操作接口。

\3. 结合上述描述，编译期与运行期都存在相应的局限性，于是最终采用了这两种方式互补来对代码进行完整扫描检测。



### 代码实现

\1. ASM实现变量及接口的hook，关于接口调用的hook方式较为常见，对于成员变量的调用方式监控这里我们通过一个示例来介绍一下。当需要扫描获取设备制造商信息Build.MANUFACTURER字段的调用时，需要进行以下几个步骤；

a、查找AOP切点代码

  **Build.MANUFACTURER**代码编译后的java字节码为：

```
  GETSTATIC android/os/Build.MANUFACTURER : Ljava/lang/String;
```



可以看出静态变量的调用所在类名**android/os/Build**，常量名为**MANUFACTURER**，常量的类型为Ljava/lang/String;，以此作为前提可对该方法进行替换操作

b、定义替换后的目标代码

  将上述代码段替换成指定的方法定义为：

```
 public static String getManufacture() {               ...               //可以在这个方法中进行相关的操作，例如调运堆栈获取等               return Build.MANUFACTURER;        }
```



c、执行替换操作

在编译期获取到字节码后，通过对字节码进行比较确定需要的替换的字节码，然后将其替换成改造后的代码，执行代码如下：

- 

```
 if (node.owner.equals("android/os/Build")             && node.name.equals("MANUFACTURER")             && node.desc.equals("Ljava/lang/String;")) {               methodNode.instructions.set(node,                     new MethodInsnNode(Opcodes.INVOKESTATIC,                     "AOPClass",    // 替换方法所在类名                     "getManufacture",  // 替换的方法名                     "()Ljava/lang/String;",false));           }
```

​    

\2. 使用epic工具实现运行期接口hook方式，这里例举一下 TelephonyManager.getSubscriberId()方法的hook方法

a、该方法一般调用方式如下



```
TelephonyManager tm =(TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);       String imsi = tm.getSubscriberId();
```



b、使用epic工具可获取到该方法运行前以及运行后的回调

```
DexposedBridge.findAndHookMethod(TelephonyManager.class, "getSubscriberId", new AOPClass());
```



c、定义目标代码，支持在调用方法的前后时机执行自定义的行为

```
public class PrivacyHook extends XC_MethodHook {             @Override         protected void beforeHookedMethod(MethodHookParam param) throws Throwable {             super.beforeHookedMethod(param);             ...         }             @Override         protected void afterHookedMethod(MethodHookParam param) throws Throwable {             super.afterHookedMethod(param);             ...         }
```

   

### 收益描述

通过AOP方式对敏感数据的访问进行跟踪，短期内定位到了违规调用的业务模块并推进整改，满足应用紧急上线需求；同时此类场景缺少有效的测试环境，使用AOP插件工具可作为一项通用验证工具推进测试工作开展。



## 场景二 : 线程使用数优化



### 背景介绍

在治理Android端APP稳定性过程中，一般都会遇到比较棘手的OOM问题，造成OOM的原因主要是java堆内存不足、虚拟内存不足、线程数量超限等内存资源使用问题；其中线程数量超限会在安卓虚拟机层抛出**java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed**异常，该异常一度在APP崩溃排行里面比较靠前，线程超限OOM问题的原因主要是APP可支配的虚拟内存有限，而在开发过程中未对多线程使用进行严格管控，很容易造成APP进程线程数超限触发OOM。



### 问题描述

对于线程数超限OOM数量线上占比过高问题，通过收集线上用户的崩溃数据，可以提取到线程数超限OOM问题发生时的线程总数、线程名称、调用堆栈这些信息；经定位归纳出以下几类问题：

- **线程池数量过多**：每个线程池都需要守护线程以及过多的空间开销，过多的线程池使用对于内存资源消耗过大；
- **常驻线程过多：** 常驻线程指的是处于waiting（等待）、blocked（阻塞）以及runnable（运行）的线程，经排查发现存在大量的常驻线程得不到释放，线程总数不断累加直到超限；
- **大量的匿名线程：** 匿名线程一般是研发最任性的一种开发方式，虽然可以实现快速、优先级最高的异步化，然而过多的匿名线程对于问题排查难度、稳定性都是一种挑战；

### 技术实施

针对已经明确的线程超限OOM的原因，开展线程使用优化的工作面临两大难点:APP中除自研业务之外还涉及一些第三方sdk，同步第三方sdk优化工作不实际；推动APP大量业务团队进行优化工作成本过高；于是考虑采用AOP这种高效而成本可控的方式进行线程使用的整体优化。

AOP需要做的工作主要包含以下几点：

- **匿名线程 ：** 将匿名线程统一命名，规则为（创建线程所在类名+线程号），并将该线程放到全局无界线程池中执行；
- **过多常驻线程** ：这里使用AOP对常用的ThreadPoolExecutor，Executors.newCachedThreadPool、Executors.newFixedThreadPool、Executors.newSingleThreadExecutor、Executors.newScheduledThreadPool等常见的5~6中线程池创建方式进行了Hook，然后插桩增加线程池的核心线程释放处理以及缩短idle线程超时设置；

### 代码实现

预编译时期增加ThreadTransformer类来遍历class文件

```java
public byte[] instrumentClass(byte[] byteArray) {
     ClassNode classNode = new ClassNode();
     ...
    ThreadTransformer threadTransformer = new ThreadTransformer(this);
     ClassNode transformClassNode = threadTransformer.transform(classNode);
    ...
    return classWriter.toByteArray();
   }
```



读取每个ClassNode来定位AOP切点

```java
while (iterator.hasNext()) {
     AbstractInsnNode node = iterator.next();


     switch (node.getOpcode()) {
         ...
        // 静态方法 可此处Hook线程池Executors.xxx的创建操作;
        case Opcodes.INVOKESTATIC:
             transformInvokeStatic((MethodInsnNode)node, classNode, methodNode);
             break;
         ...
        // 此处可Hook匿名线程Thread的new方法
         case Opcodes.NEW:
             transformNew((TypeInsnNode)node, classNode, methodNode);
             break;
         ...
   }
```

  
代码插桩，替换为优化后的方法

```java

 //代码替换
 private static final String EXECUTORS_CLASS = "java/util/concurrent/Executors";
 private void transformInvokeStatic(MethodInsnNode node, ClassNode classNode, MethodNode methodNode) {
   if (node.owner.equals(EXECUTORS_CLASS)) {
       switch (node.name) {
           case "newCachedThreadPool":

          case "newFixedThreadPool":
           case "newSingleThreadExecutor":
           case "newSingleThreadScheduledExecutor":
           case "newScheduledThreadPool":
           case "newWorkStealingPool":
               ...
               node.owner = EXECUTORS_AOP_CLASS;
               node.desc = desc1;


               methodNode.instructions.insertBefore(node, new LdcInsnNode(makeThreadName(classNode.name)));
               break;
           ...
         }
     }
 }
```

举例对Executors.newFixedThreadPool()方法替换为：

```java
public static ExecutorService newFixedThreadPool(final int nThreads, final String name) {
        if (optimizedExecutorsEnabled) {
            return newOptimizedFixedThreadPool(nThreads, name);
        }
        return Executors.newFixedThreadPool(nThreads, new NamedThreadFactory(name));
      }
    
      //优化后的方法
      public static ExecutorService newOptimizedFixedThreadPool(final int nThreads, final String name) {
         ThreadPoolExecutor executor = new ThreadPoolExecutor(nThreads, nThreads,
                 0L, TimeUnit.MILLISECONDS,
                 new LinkedBlockingQueue<Runnable>(),
                 new NamedThreadFactory(name));
         executor.setKeepAliveTime(DEFAULT_KEEP_ALIVE, TimeUnit.MILLISECONDS);
         executor.allowCoreThreadTimeOut(true);
         return executor;
       }

```


### 收益描述

通过AOP进行匿名线程、常驻线程优化之后，java线程数在运行期均值从190条降到140条左右，优化效果30%左右， 灰度期间线上监控平台检测到的线程超限OOM的数量减少50%，优化效果明显。

# 写在最后

通过AOP技术开发gradle插件工具已经在集团内部落地多个应用场景，在高可用性以及研发提效方面取得了一定的成绩，然而局限性就是存在一定的技术门槛，后续需要做到的是尽量的通用化、配置化，同时新的技术也在尝试实践，比如虚拟APP技术，最终目标是发挥最大效能基础上将接入成本降到极致。



# 参考

https://mp.weixin.qq.com/s/Gxh_kI7uhg0JnxwFD1Om4A

https://juejin.cn/post/6983513030331990029