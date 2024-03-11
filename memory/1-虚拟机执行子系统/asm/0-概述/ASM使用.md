# 工具



## 插件：ASM Bytecode Viewer

https://plugins.jetbrains.com/plugin/5918-asm-bytecode-outline

### 去除无用信息

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20201021184825.png)

## javap

假设有下面这样的一个类

```
public class DemoClass {
	public static void main(String[] args) {
		System.out.println();
	}
}
```

通过javap操作

```
$ javap -c classtest.DemoClass
Compiled from "DemoClass.java"
public class classtest.DemoClass extends java.lang.Object{
public classtest.DemoClass();
  Code:
   0:   aload_0
   1:   invokespecial   #8; //Method java/lang/Object."<init>":()V
   4:   return

public static void main(java.lang.String[]);
  Code:
   0:   getstatic       #16; //Field java/lang/System.out:Ljava/io/PrintStream;
   3:   invokevirtual   #22; //Method java/io/PrintStream.println:()V
   6:   return

}
```



扫描这个类的方法

```
public class DemoClassTest {
    public static void main(String[] args) throws IOException {
        ClassReader cr = new ClassReader(DemoClass.class.getName());
        cr.accept(new DemoClassVisitor(), ClassReader.SKIP_DEBUG);
        System.out.println("---ALL END---");
    }
}
class DemoClassVisitor extends ClassVisitor {
    public DemoClassVisitor() {
        super(Opcodes.ASM4);
    }
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        System.out.println("at Method " + name);
        //
        MethodVisitor superMV = super.visitMethod(access, name, desc, signature, exceptions);
        return new DemoMethodVisitor(superMV, name);
    }
}
class DemoMethodVisitor extends MethodVisitor {
    private String methodName;
    public DemoMethodVisitor(MethodVisitor mv, String methodName) {
        super(Opcodes.ASM4, mv);
        this.methodName = methodName;
    }
    public void visitCode() {
        System.out.println("at Method ‘" + methodName + "’ Begin...");
        super.visitCode();
    }
    public void visitEnd() {
        System.out.println("at Method ‘" + methodName + "’End.");
        super.visitEnd();
    }
}
```

1. 上面这段程序我们首先在第三行使用 ClassReader 去读取 DemoClass 类的字节码信息。
2. 其次通过“cr.accept(new DemoClassVisitor(), ClassReader.SKIP_DEBUG);”方法开始Visitor扫描整个字节码。
3. SKIP_DEBUG选项的意义是在扫描过程中略过所有有关行号方面的内容。
4. 在DemoClassVisitor类中我们重写了visitMethod方法，当遇到方法的时候打印出方法名。
5. 随后我们返回DemoMethodVisitor对象，用以输出方法的开始和结束

```
at Method <init>
at Method ‘<init>’ Begin...
at Method ‘<init>’End.
at Method main
at Method ‘main’ Begin...
at Method ‘main’End.
---ALL END---
```



## ASMifier

[ASM官网](https://asm.ow2.io/#Q10) 下载 `asm-all.jar` 库

```java
java -classpath asm-all-5.2.jar org.objectweb.asm.util.ASMifier Demo.class 
```



![image-20201123145215805](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20201123145216.png)





# ASM库结构



![image-20201021153814782](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20201021153814.png)

- Core：为其他包提供基础的读、写、转化Java字节码和定义的API，并且可以生成Java字节码和实现大部分字节码的转换，在 [访问者模式和 ASM](https://www.jianshu.com/p/e4b8cb0b3204) 中介绍的几个重要的类就在 Core API 中：ClassReader、ClassVisitor 和 ClassWriter 类.

- Tree：提供了 Java 字节码在内存中的表现

- Commons：提供了一些常用的简化字节码生成、转换的类和适配器

- Util：包含一些帮助类和简单的字节码修改类，有利于在开发或者测试中使用

- XML：提供一个适配器将XML和SAX-comliant转化成字节码结构，可以允许使用XSLT去定义字节码转化





# 指令

ALOAD_0：
  这个指令是LOAD系列指令中的一个，它的意思表示装载当前第 0 个元素到堆栈中。代码上相当于“this”。而这个数据元素的类型是一个引用类型。这些指令包含了：ALOAD，ILOAD，LLOAD，FLOAD，DLOAD。区分它们的作用就是针对不用数据类型而准备的LOAD指令，此外还有专门负责处理数组的指令 SALOAD。

invokespecial：
  这个指令是调用系列指令中的一个。其目的是调用对象类的方法。后面需要给上父类的方法完整签名。“#8”的意思是 .class 文件常量表中第8个元素。值为：“java/lang/Object."<init>":()V”。结合ALOAD_0。这两个指令可以翻译为：“super()”。其含义是调用自己的父类构造方法。

GETSTATIC：
  这个指令是GET系列指令中的一个其作用是获取静态字段内容到堆栈中。这一系列指令包括了：GETFIELD、GETSTATIC。它们分别用于获取动态字段和静态字段。

IDC：
  这个指令的功能是从常量表中装载一个数据到堆栈中。

invokevirtual：
  也是一种调用指令，这个指令区别与 invokespecial 的是它是根据引用调用对象类的方法。这里有一篇文章专门讲解这两个指令：“http://wensiqun.iteye.com/blog/1125503”。

RETURN：
  这也是一系列指令中的一个，其目的是方法调用完毕返回：可用的其他指令有：IRETURN，DRETURN，ARETURN等，用于表示不同类型参数的返回。



## 例子

拦截器代码如下

```
public class AopInterceptor {
    public static void beforeInvoke() {
        System.out.println("before");
    }
    public static void afterInvoke() {
        System.out.println("after");
    }
}
```

最终执行的代码

```
public class TestBean {
    public void halloAop() {
        AopInterceptor.beforeInvoke();
        System.out.println("Hello Aop");
        AopInterceptor.afterInvoke();
    }
}
```



## 操作流程

1. 需要创建一个 ClassReader 对象，将 .class 文件的内容读入到一个字节数组中

2. 然后需要一个 ClassWriter 的对象将操作之后的字节码的字节数组回写

3. 需要事件过滤器 ClassVisitor。在调用 ClassVisitor 的某些方法时会产生一个新的 XXXVisitor 对象，当我们需要修改对应的内容时只要实现自己的 XXXVisitor 并返回就可以了



### 1 ASM-Bytecode转换结果



```
{
mv = cw.visitMethod(ACC_PUBLIC, "halloAop", "()V", null, null);
mv.visitCode();
Label l0 = new Label();
mv.visitLabel(l0);
mv.visitLineNumber(24, l0);
mv.visitMethodInsn(INVOKESTATIC, "org/more/test/asm/AopInterceptor", "beforeInvoke", "()V");
Label l1 = new Label();
mv.visitLabel(l1);
mv.visitLineNumber(25, l1);
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("Hello Aop");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V");
Label l2 = new Label();
mv.visitLabel(l2);
mv.visitLineNumber(26, l2);
mv.visitMethodInsn(INVOKESTATIC, "org/more/test/asm/AopInterceptor", "afterInvoke", "()V");
Label l3 = new Label();
mv.visitLabel(l3);
mv.visitLineNumber(27, l3);
mv.visitInsn(RETURN);
Label l4 = new Label();
mv.visitLabel(l4);
mv.visitLocalVariable("this", "Lorg/more/test/asm/TestBean;", null, l0, l4, 0);
mv.visitMaxs(2, 1);
mv.visitEnd();
}
```



上面生成的代码中 4，5，6，8，9，10，14，15，16，18，19，20 行看到如下内容：

```
Label l2 = new Label();
mv.visitLabel(l2);
mv.visitLineNumber(26, l2);
```

这些内容表示 Java 代码的行号标记，可以删除不用。在方法的最后部分代码：

```
Label l4 = new Label();
mv.visitLabel(l4);
mv.visitLocalVariable("this", "Lorg/more/test/asm/TestBean;", null, l0, l4, 0);
```

也是可以被删除不用的，这部分代码表示向 class 文件中写入方法本地变量表的名称以及类型。经过精简之后就是下面的代码了：

```
mv = cw.visitMethod(ACC_PUBLIC, "halloAop", "()V", null, null);
mv.visitCode();
mv.visitMethodInsn(INVOKESTATIC, "org/more/test/asm/AopInterceptor", "beforeInvoke", "()V");
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("Hello Aop");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V");
mv.visitMethodInsn(INVOKESTATIC, "org/more/test/asm/AopInterceptor", "afterInvoke", "()V");
mv.visitInsn(RETURN);
mv.visitMaxs(2, 1);
mv.visitEnd();
```

逐句对应解释：
  01 行：相当于 public void halloAop() 方法声明。
  02 行：正式开发方法内容的填充。
  03 行：调用静态方法，相当于：“AopInterceptor.beforeInvoke();”。
  04 行：取得一个静态字段将其放入堆栈，相当于“System.out”。“Ljava/io/PrintStream;”是字段类型的描述，翻译过来相当于：“java.io.PrintStream”类型。在字节码中凡是引用类型均由“L”开头“;”结束表示，中间是类型的完整名称。
  05 行：将字符串“Hello Aop”放入堆栈，此时堆栈中第一个元素是“System.out”，第二个元素是“Hello Aop”。
  06 行：调用PrintStream类型的“println”方法。签名“(Ljava/lang/String;)V”表示方法需要一个字符串类型的参数，并且无返回值。
  07 行：调用静态方法，相当于：“AopInterceptor.afterInvoke();”。
  08 行：是 JVM 在编译时为方法自动加上的“return”指令。该指令必须在方法结束时执行不可缺少。
  09 行：表示在执行这个方法期间方法的堆栈空间最大给予多少。
  10 行：表示方法输出结束。



### 2 Aop 实现的 ASM 代码



```
class AopClassAdapter extends ClassVisitor implements Opcodes {
    public AopClassAdapter(int api, ClassVisitor cv) {
        super(api, cv);
    }
    public void visit(int version, int access, String name,
                         String signature, String superName, String[] interfaces) {
        //更改类名，并使新类继承原有的类。
        super.visit(version, access, name + "_Tmp", signature, name, interfaces);
        {//输出一个默认的构造方法
            MethodVisitor mv = super.visitMethod(ACC_PUBLIC, "<init>",
                          "()V", null, null);
            mv.visitCode();
            mv.visitVarInsn(ALOAD, 0);
            mv.visitMethodInsn(INVOKESPECIAL, name, "<init>", "()V");
            mv.visitInsn(RETURN);
            mv.visitMaxs(1, 1);
            mv.visitEnd();
        }
    }
    public MethodVisitor visitMethod(int access, String name,
                             String desc, String signature, String[] exceptions) {
        if ("<init>".equals(name))
            return null;//放弃原有类中所有构造方法
        if (!name.equals("halloAop"))
            return null;// 只对halloAop方法执行代理
        //
        MethodVisitor mv = super.visitMethod(access, name,
                                          desc, signature, exceptions);
        return new AopMethod(this.api, mv);
    }
}
class AopMethod extends MethodVisitor implements Opcodes {
    public AopMethod(int api, MethodVisitor mv) {
        super(api, mv);
    }
    public void visitCode() {
        super.visitCode();
        this.visitMethodInsn(INVOKESTATIC, "org/more/test/asm/AopInterceptor", "beforeInvoke", "()V");
    }
    public void visitInsn(int opcode) {
        if (opcode == RETURN) {//在返回之前安插after 代码。
            mv.visitMethodInsn(INVOKESTATIC, "org/more/test/asm/AopInterceptor", "afterInvoke", "()V");
        }
        super.visitInsn(opcode);
    }
}

```



### 3 ASM 改写 Java 类

```
ClassWriter cw = new ClassWriter(0);
//
InputStream is = Thread.currentThread().getContextClassLoader()
          .getResourceAsStream("org/more/test/asm/TestBean.class");
ClassReader reader = new ClassReader(is);
reader.accept(new AopClassAdapter(ASM4, cw), ClassReader.SKIP_DEBUG);
//
byte[] code = cw.toByteArray();
```



## 完整代码

### 目标类 

```
public class TestBean {
    public void halloAop() {
        System.out.println("Hello Aop");
    }
}
```



### Aop拦截器处理程序 

```
public class AopInterceptor {
    public static void beforeInvoke() {
        System.out.println("before");
    };
    public static void afterInvoke() {
        System.out.println("after");
    };
}
```

### 生成字节码并负责转换为Class类型的类装载器

```
class AopClassLoader extends ClassLoader implements Opcodes {
    public AopClassLoader(ClassLoader parent) {
        super(parent);
    }
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        if (!name.contains("TestBean_Tmp"))
            return super.loadClass(name);
        try {
            ClassWriter cw = new ClassWriter(0);
            //
            InputStream is = Thread.currentThread().getContextClassLoader().getResourceAsStream("org/more/test/asm/TestBean.class");
            ClassReader reader = new ClassReader(is);
            reader.accept(new AopClassAdapter(ASM4, cw), ClassReader.SKIP_DEBUG);
            //
            byte[] code = cw.toByteArray();
            //            FileOutputStream fos = new FileOutputStream("c:\\TestBean_Tmp.class");
            //            fos.write(code);
            //            fos.flush();
            //            fos.close();
            return this.defineClass(name, code, 0, code.length);
        } catch (Throwable e) {
            e.printStackTrace();
            throw new ClassNotFoundException();
        }
    }
}
```

### ASM 改写字节码的完整程序 

```
class AopClassAdapter extends ClassVisitor implements Opcodes {
    public AopClassAdapter(int api, ClassVisitor cv) {
        super(api, cv);
    }
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        //更改类名，并使新类继承原有的类。
        super.visit(version, access, name + "_Tmp", signature, name, interfaces);
        {
            MethodVisitor mv = super.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
            mv.visitCode();
            mv.visitVarInsn(ALOAD, 0);
            mv.visitMethodInsn(INVOKESPECIAL, name, "<init>", "()V");
            mv.visitInsn(RETURN);
            mv.visitMaxs(1, 1);
            mv.visitEnd();
        }
    }
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        if ("<init>".equals(name))
            return null;
        if (!name.equals("halloAop"))
            return null;
        //
        MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);
        return new AopMethod(this.api, mv);
    }
}
class AopMethod extends MethodVisitor implements Opcodes {
    public AopMethod(int api, MethodVisitor mv) {
        super(api, mv);
    }
    public void visitCode() {
        super.visitCode();
        this.visitMethodInsn(INVOKESTATIC, "org/more/test/asm/AopInterceptor", "beforeInvoke", "()V");
    }
    public void visitInsn(int opcode) {
        if (opcode == RETURN) {
            mv.visitMethodInsn(INVOKESTATIC, "org/more/test/asm/AopInterceptor", "afterInvoke", "()V");
        }
        super.visitInsn(opcode);
    }
}
```





# ClassVisitor

具体使用文档：

https://www.yuque.com/mikaelzero/asm/xzz9en



## 树形结构

```
Class
    Annotation
        Annotation
            ...
    Field
        Annotation
            ...
    Method
        Annotation
            ...
```



| 树形关系   | 使用的接口        |
| ---------- | :---------------- |
| Class      | ClassVisitor      |
| Field      | FieldVisitor      |
| Method     | MethodVisitor     |
| Annotation | AnnotationVisitor |



## ClassVisitor结构

![image-20201123143810976](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20201123143903.png)

### visit(int , int , String , String , String , String[])

该方法是当扫描类时第一个拜访的方法，主要用于类声明使用。下面是对方法中各个参数的示意：**visit( 类版本 ,** **修饰符** **, 类名 , 泛型信息 , 继承的父类 , 实现的接口****)**。例如：

```
public class TestBean {

等价于：
visit(V1_6, ACC_PUBLIC | ACC_SUPER , "org/more/test/asm/simple/TestBean",
      null, "java/lang/Object", null)
```

  **第一个参数：**表示类版本：V1_6，表示 “.class” 文件的版本是 JDK 1.6。可用的其他版本有：V1_1（JRE_1.1）、V1_2（J2SE_1.2）、V1_3（J2SE_1.3）、V1_4（J2SE_1.4）、V1_5（J2SE_1.5）、V1_6（JavaSE_1.6）、V1_7（JavaSE_1.7）。我们所指的 JDK 6 或 JDK 7 实际上就是只 JDK 1.6 或 JDK 1.7。

  **第二个参数：**表示类的修饰符：修饰符在 ASM 中是以 “ACC_” 开头的常量进行定义。可以作用到类级别上的修饰符有：ACC_PUBLIC（public）、ACC_PRIVATE（private）、ACC_PROTECTED（protected）、ACC_FINAL（final）、ACC_SUPER（extends）、ACC_INTERFACE（接口）、ACC_ABSTRACT（抽象类）、ACC_ANNOTATION（注解类型）、ACC_ENUM（枚举类型）、ACC_DEPRECATED（标记了@Deprecated注解的类）、ACC_SYNTHETIC。

  **第三个参数：**表示类的名称：通常我们的类完整类名使用 “org.test.mypackage.MyClass” 来表示，但是到了字节码中会以路径形式表示它们 “org/test/mypackage/MyClass” 值得注意的是虽然是路径表示法但是不需要写明类的 “.class” 扩展名。

  **第四个参数：**表示泛型信息，如果类并未定义任何泛型该参数为空。Java 字节码中表示泛型时分别对接口和类采取不同的定义。该参数的内容格式如下：

```
<泛型名:基于的类型....>Ljava/lang/Object;

<泛型名::基于的接口....>Ljava/lang/Object;
```

其中 “泛型名:基于的类型” 内容可以无限的写下去，例如：

```
public class TestBean<T,V,Z> {

泛型参数为：<T:Ljava/lang/Object;V:Ljava/lang/Object;Z:Ljava/lang/Object;>Ljava/lang/Object;
分析结构如下：
  <
   T:Ljava/lang/Object;
   V:Ljava/lang/Object;
   Z:Ljava/lang/Object;
  >
   Ljava/lang/Object;
```

再或者：

```
public class TestBean<T extends Date, V extends ArrayList> {

泛型参数为：<T:Ljava/util/Date;V:Ljava/util/ArrayList;>Ljava/lang/Object;
分析结构如下：
  <
   T:Ljava/util/Date;
   V:Ljava/util/ArrayList;
  >
   Ljava/lang/Object;
```

  以上内容只是针对泛型内容是基于某个具体类型的情况，如果泛型是基于接口而非类型则定义方式会有所不同，这一点需要注意。例如：

```
public class TestBean<T extends Serializable, V> {

泛型参数为：<T::Ljava/io/Serializable;V:Ljava/lang/Object;>Ljava/lang/Object;
分析结构如下：
  <
   T::Ljava/io/Serializable; //比类型多出一个“:”
   V:Ljava/lang/Object;
  >
   Ljava/lang/Object;
```

  **第五个参数：**表示所继承的父类。由于 Java 的类是单根结构，即所有类都继承自 java.lang.Object 因此可以简单的理解为任何类都会具有一个父类。虽然在编写 Java 程序时我们没有去写 extends 关键字去明确继承的父类，但是 JDK在编译时 总会为我们加上 “ extends Object”。所以倘若某一天你看到这样一份代码也不要过于紧张。

  **第六个参数：**表示类实现的接口，在 Java 中类是可以实现多个不同的接口因此此处是一个数组例如：

```
public class TestBean implements Serializable , List {

该参数会以 “[java/io/Serializable, java/util/List]” 形式出现。
```

  这里需要补充一些内容，如果类型其本身就是接口类型。对于该方法而言，接口的父类类型是 “java/lang/Object”，接口所继承的所有接口都会出现在第六个参数中。例如：

```
public inteface TestBean implements Serializable , List {

最后两个参数对应为:
    "java/lang/Object", ["java/io/Serializable","java/util/List"]
```





### visitField(int access, String name, String desc, String signature, Object value)

visitField(修饰符 , 字段名 , 字段类型 , 泛型描述 , 默认值)



### visitAnnotation(String , boolean)

该方法是当扫描器扫描到类注解声明时进行调用。下面是对方法中各个参数的示意：**visitAnnotation(注解类型 , 注解是否可以在 JVM 中可见)**。例如：

```
@Bean({ "" })
public class TestBean {

@Bean等价于：
    visitAnnotation("Lnet/hasor/core/gift/bean/Bean;", true);
```

下面是 @Bean 的源代码：

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE })
public @interface Bean {
    /** Bean名称。*/
    public String[] value();
}
```

  **第一个参数：**表示的是，注解的类型。它使用的是（“L” + “类型路径” + “;”）形式表述。

  **第二个参数：**表示的是，该注解是否在 JVM 中可见。这个参数的具体含义可以理解为：如果为 true 表示虚拟机可见，我们可以通过下面这样的代码获取到注解类型：

```
testBeanType.getAnnotation(TestAnno.class);
```

  谈到这里就需要额外说明一下在声明注解时常见的 “@Retention(RetentionPolicy.RUNTIME)” 标记。RetentionPolicy 是一个枚举它具备三个枚举元素其每个含义可以理解为：

  1.RetentionPolicy.SOURCE：声明注解只保留在 Java 源程序中，在编译 Java 类时注解信息不会被写入到 Class。如果使用的是这个配置 ASM 也将无法探测到这个注解。
  2.RetentionPolicy.CLASS：声明注解仅保留在 Class 文件中，JVM 运行时并不会处理它，这意味着 ASM 可以在 visitAnnotation 时候探测到它，但是通过Class 反射无法获取到注解信息。
  3.RetentionPolicy.RUNTIME：这是最常用的一种声明，ASM 可以探测到这个注解，同时 Java 反射也可以取得注解的信息。所有用到反射获取的注解都会用到这个配置，就是这个原因。

**3.visitField(int , String , String , String , Object)
**  该方法是当扫描器扫描到类中字段时进行调用。下面是对方法中各个参数的示意：**visitField(修饰符 , 字段名 , 字段类型 , 泛型描述 , 默认值)**。例如：

```
public class TestBean {
    private String stringData;

stringData字段等价于：
    visitField(ACC_PRIVATE, "stringData", "Ljava/lang/String;", null, null)
```

  **第一个参数：**表示字段的修饰符，修饰符在 ASM 中是以 “ACC_” 开头的常量进行定义。可以作用到字段级别上的修饰符有：ACC_PUBLIC（public）、ACC_PRIVATE（private）、ACC_PROTECTED（protected）、ACC_STATIC（static）、ACC_FINAL（final）、ACC_VOLATILE（volatile）、ACC_TRANSIENT（transient）、ACC_ENUM（枚举）、ACC_DEPRECATED（标记了@Deprecated注解的字段）、ACC_SYNTHETIC。

  **第二个参数：**表示字段的名称。



  **第三个参数：**表示字段的类型，其格式为：（“L” + 类型路径 + “;”）。

   **第四个参数：**表示泛型信息， 泛型类型描述是使用（“T” + 泛型名 + “;”）加以说明。例如：“private T data;” 字段的泛型描述将会是 “ TT; ”， “ private V data; ” 字段的泛型描述将会是 “ TV; ”。泛型名称将会在 “visit(int , int , String , String , String , String[])” 方法中第五个参数中加以表述。例如：`public class TestBean<T, V> {    private T data;    private V value; 等价于： visit(V1_6, ACC_PUBLIC | ACC_SUPER , "org/more/test/asm/simple/TestBean",      "<T:Ljava/lang/Object;V:Ljava/lang/Object;>Ljava/lang/Object;", //定义了两个泛型类型 T 和 V      "java/lang/Object", null) visitField(ACC_PRIVATE, "data", "Ljava/lang/Object;", "TT;", null)  //data 泛型名称为 T visitField(ACC_PRIVATE, "value", "Ljava/lang/Object;", "TV;", null) // value 泛型名称为 V`

还有一种情况，倘若类在定义泛型时候已经基于某个类型那么生成的代码将会是如下形式：

```
public class TestBean<T extends Serializable, V> {
    private T data;
    private V value;

等价于：

visit(V1_6, ACC_PUBLIC | ACC_SUPER , "org/more/test/asm/simple/TestBean",
      "<T::Ljava/io/Serializable;V:Ljava/lang/Object;>Ljava/lang/Object;", //定义了两个泛型类型 T 和 V
      "java/lang/Object", null)

visitField(ACC_PRIVATE, "data", "Ljava/io/Serializable;", "TT;", null)  //data 泛型名称为 T
visitField(ACC_PRIVATE, "value", "Ljava/lang/Object;", "TV;", null)     // value 泛型名称为 V
```

  **第五个参数：**表示的是默认值， 由于默认值是 Object 类型大家可能以为可以是任何类型。这里要澄清一下，默认值中只能用来表述 Java 基本类型这其中包括了（byte、sort、int、long、float、double、boolean、String）其他所有类型都不不可以进行表述。并且只有标有 “final” 修饰符的字段并且该字段赋有初值时这个参数才会有值。例如类：`public class TestBean {    private final String data;    public TestBean() {        data = "aa";    } ....`

在执行 “visitField” 方法时候，这个参数的就是 null 值，下面这种代码也会是 null 值：

```
public class TestBean {
    private final Date data;
    public TestBean() {
        data =new Date();
    }
....
```

此外如果字段使用的是基本类型的包装类型，诸如：Integer、Long...也会为空值：

```
public class TestBean {
    private final Integer intData = 12;
...
```

能够正确得到默认值的代码应该是这个样子的：

```
public class TestBean {
    private final String data    = "ABC";
    private final int    intData = 12;
...
```



### visitMethod

该方法是当扫描器扫描到类的方法时进行调用。下面是对方法中各个参数的示意：**visitMethod(修饰符 , 方法名 , 方法签名 , 泛型信息 , 抛出的异常)**。例如：

```
public class TestBean {
    public int halloAop(String param) throws Throwable {

等价于：

visit(V1_6, ACC_PUBLIC | ACC_SUPER , "org/more/test/asm/simple/TestBean",
      null, "java/lang/Object", null)

visitMethod(ACC_PUBLIC, "<init>", "()V", null, null)
visitMethod(ACC_PUBLIC, "halloAop", "(Ljava/lang/String;)I", null, [java/lang/Throwable])
```

  **第一个参数：**表示方法的修饰符，修饰符在 ASM 中是以 “ACC_” 开头的常量进行定义。可以作用到方法级别上的修饰符有：ACC_PUBLIC（public）、ACC_PRIVATE（private）、ACC_PROTECTED（protected）、ACC_STATIC（static）、ACC_FINAL（final）、ACC_SYNCHRONIZED（同步的）、ACC_VARARGS（不定参数个数的方法）、ACC_NATIVE（native类型方法）、ACC_ABSTRACT（抽象的）、ACC_DEPRECATED（标记了@Deprecated注解的方法）、ACC_STRICT、ACC_SYNTHETIC。

  **第二个参数：**表示方法名，在 ASM 中 “visitMethod” 方法会处理（构造方法、静态代码块、私有方法、受保护的方法、共有方法、native类型方法）。在这些范畴中构造方法的方法名为 “<init>”，静态代码块的方法名为 “<clinit>”。列如：

```
public class TestBean {
    public int halloAop(String param) throws Throwable {
        return 0;
    }
    static {
        System.out.println();
    }
...

等价于：

visit(V1_6, ACC_PUBLIC | ACC_SUPER , "org/more/test/asm/simple/TestBean",
      null, "java/lang/Object", null)

visitMethod(ACC_PUBLIC, "<clinit>", "()V", null, null)
visitMethod(ACC_PUBLIC, "<init>", "()V", null, null)
visitMethod(ACC_PUBLIC, "halloAop", "(Ljava/lang/String;)I", null, [java/lang/Throwable])
```

  **第三个参数：**表示方法签名，方法签名的格式如下：“(参数列表)返回值类型”。在字节码中不同的类型都有其对应的代码，如下所示：

```
"I"        = int
"B"        = byte
"C"        = char
"D"        = double
"F"        = float
"J"        = long
"S"        = short
"Z"        = boolean
"V"        = void
"[...;"    = 数组
"[[...;"   = 二维数组
"[[[...;"  = 三维数组
"L....;"   = 引用类型
```

  下面是一些方法签名对应的方法参数列表。

| String[]                              | [Ljava/lang/String;                                          |
| ------------------------------------- | ------------------------------------------------------------ |
| String[][]                            | [[Ljava/lang/String;                                         |
| int, String, String, String, String[] | ILjava/lang/String;Ljava/lang/String;Ljava/lang/String;[Ljava/lang/String; |
| int, boolean, long, String[], double  | IZJ[Ljava/lang/String;D                                      |
| Class<?>, String, Object... paramType | Ljava/lang/Class;Ljava/lang/String;[Ljava/lang/Object;       |
| int[]                                 | [I                                                           |

  **第四个参数：**凡是具有泛型信息的方法，该参数都会有值。并且该值的内容信息基本等于第三个参数的拷贝，只不过不同的是泛型参数被特殊标记出来。例如：

```
public class TestBean<T, V extends List> {
    public T halloAop(V abc, int aaa) throws Throwable {

方法签名：(Ljava/util/List;I)Ljava/lang/Object;
泛型签名：(TV;I)TT;
public class TestBean<T, V extends List> {
    public String halloAop(V abc, int aaa) throws Throwable {

方法签名：(Ljava/util/List;I)Ljava/lang/String;
泛型签名：(TV;I)Ljava/lang/String;
```

  可以看出泛型信息中用于标识泛型类型的结构是（“T” + 泛型名 + “;”），还有一种情况就是。泛型是声明在方法上。例如：

```
public class TestBean {
    public <T extends List> String halloAop(T abc, int aaa) throws Throwable {

方法签名：(Ljava/util/List;I)Ljava/lang/String;
泛型签名：<T::Ljava/util/List;>(TT;I)Ljava/lang/String; //泛型类型基于接口
public class TestBean {
    public <T> String halloAop(T abc, int aaa) throws Throwable {

方法签名：(Ljava/lang/Object;I)Ljava/lang/String;
泛型签名：<T:Ljava/lang/Object;>(TT;I)Ljava/lang/String; //泛型类型基于类型
```

  **第五个参数：**用来表示将会抛出的异常，如果方法不会抛出异常。则该参数为空。这个参数的表述形式比较简单，举一个例子：

```
public class TestBean {
    public <T> String halloAop(T abc, int aaa) throws Throwable,Exception {

异常参数为：[java/lang/Throwable, java/lang/Exception]
```



### visitEnd()

该方法是当扫描器完成类扫描时才会调用，如果想在类中追加某些方法。可以在该方法中实现。在后续文章中我们会用到这个方法。

  提示：ACC_SYNTHETIC、ACC_STRICT：这两个修饰符我也不清楚具体功能含义是什么，如果有知道的朋友还望补充说明



### 





# MethodVisitor



![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20201023164746.png)



### 重点函数

```
MethodVisitor.visitCode();
MethodVisitor.visitMaxs(maxStack, maxLocals);
MethodVisitor.visitEnd();
```

- 第一个方法：表示ASM开始扫描这个方法。
- 第二个方法：该方法是visitEnd之前调用的方法，可以反复调用。用以确定类方法在执行时候的堆栈大小。
- 第三个方法：表示方法输出完毕



### 构造方法

  关于方法名或许读者注意到了在扫描这个类的时候，有一个特殊的方法被扫描到了“<init>”，这个方法是传说中的构造方法。当Java在编译的时候没有发现类文件中有构造方法的定义会为其创建一个默认的无参构造方法。这个“<init>”就是那个由系统添加的构造方法。现在我们为类填写一个构造方法如下：

```
public class DemoClass {
    public DemoClass(int a) {
        
    }
    public static void main(String[] args) {
        System.out.println();
    }
}
```

  再次扫描这个类，你会发现它的结果和刚才是一样的，这是由于我们编写的构造方法替换了系统默认生成的那个。

### 静态代码块

  在Class我们接触过用“static {  }”包含的代码，这个是我们常说的静态代码块。这个代码快ASM在扫描字节码的时候也会遇到它，大家可千万别以为这真的是一个什么代码块。所有的静态代码快最后都会放到“<clinit>”方法中。

  静态代码快只有一个，现有下面这个的一个类。在编写这个类的时候我有意的写了两个不同的静态代码块的类：

```
public class DemoClass {
    static {
        int a;
        System.out.println(11);
    }
    public static void main(String[] args) {
        System.out.println();
    }
    static {
        int a;
        System.out.println(22);
    }
}
```

  ASM在扫描这个类的时候你会发现虽然类中存在多个静态代码快，但是最后类文件中只会出现了一个“<clinit>”方法。JVM在编译Class的时候估计已经将多个静态代码块合并到一起了。



# AdviceAdapter



## 结构

![image-20201021154152163](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20201021154152.png)

## 重点函数

void visitCode()：表示 ASM 开始扫描这个方法

void onMethodEnter()：进入这个方法

void onMethodExit()：即将从这个方法出去

void onVisitEnd()：表示方法扫码完毕



# FieldVisitor

