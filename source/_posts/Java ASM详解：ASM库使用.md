---
title: Java ASM详解：ASM库使用
date: 2021-02-13
updated: 2021-08-30 12:23:13
category:
	- Java
tags:
	- Java
	- ASM
toc: true
comment: true
---

几个月之后，终于来到了ASM的第二篇专栏（指直接咕了半年）

这篇专栏主要说一说ASM库怎么用，电脑端观看更佳。

## 读取的起源：ClassReader

ClassReader位于`org.objectweb.asm`包下（基础类都在这个包），它是读取字节码的开始，通过它我们才能进行字节码解析。

首先是构造函数：

```Java
public ClassReader(byte[] classFile)
public ClassReader(byte[] classFileBuffer, int classFileOffset, int classFileLength)
ClassReader(byte[] classFileBuffer, int classFileOffset, boolean checkClassVersion)
public ClassReader(InputStream inputStream) throws IOException
public ClassReader(String className) throws IOException
```

这几种构造函数都是以传入数据为目标的：前两个，根据传入的byte数组（第二个指定了偏移量和长度）解析类；倒数第二个，通过`InputStream`传入；倒数第一个，根据类的全限定名获得对象；中间的则是不开放的API，可以忽略。下面是使用的例子：

```Java
ClassReader reader = new ClassReader("com/github/nickid2018/asm/TestClass");
ClassReader reader2 = new ClassReader(classBytes, 0, 3370);
ClassReader reader3 = new ClassReader(inputStreamClassFile);
```

说完了对象的构建，下面是它的用法。它最重要的方法是`accept`，其余的方法基本用不上（都内部自己用的）

```Java
public void accept(ClassVisitor classVisitor, int parsingOptions)
public void accept(ClassVisitor classVisitor, Attribute[] attributePrototypes, int parsingOptions)
```

先抛开`Attribute[]`这个参数，这个以后可能会说。第一个参数ClassVisitor是你要传入的访问器：ASM整体是**Visitor设计模式**。最后一个参数int是代表读取模式，它有4个基本取值，这些值可以被or（|）连接：

| 常量值 | 读取方式 |
| :----: | :----- |
| SKIP_CODE | 跳过代码属性 |
| SKIP_DEBUG | 跳过源文件、局部变量表、局部变量类型表、方法参数列表、行号 |
| SKIP_FRAME | 跳过帧（visitFrame），帧是JVM验证类阶段使用的数据 |
| EXPANDS_FRAMES | 扩展堆栈映射帧 |

下面是例子：

```Java
ClassVisitor cv = ...;
classReader.accept(cv, 0);
classReader.accept(cv, ClassWriter.SKIP_CODE);
```

关于ClassReader的使用到这里差不多结束了，下面先讲一下访问标志，然后再说ClassVisitor等类。

## 访问标志（Access Flag）

访问标志是用于JVM访问类、字段、方法检查和调用的一个int。这些标志既包含了我们常见的public这种访问限定符，还包含了static、final这种修饰符，除此之外还有声明类为接口的interface，为枚举的enum。

完整的访问标志如下表（省略了前缀ACC_）：

| Access Flag | 常量值 | 修饰目标 | 用途 |
| :----:      | :-:   | :-:     | :--- |
| PUBLIC | 0x0001 | class, field, method | 访问限定，公开 |
| PRIVATE | 0x0002 | class, field, method | 访问限定，私密 |
| PROTECTED | 0x0004 | class, field, method | 访问限定，受保护 |
| STATIC | 0x0008 | field, method | 静态 |
| FINAL | 0x0010 | class, field, method, parameter | 对于类为不可继承，对于其他为不可修改 |
| SUPER | 0x0020 | class | 调用invokespecial时会特殊处理超类方法 |
| SYNCHRONIZED | 0x0020 | method | 同步方法 |
| OPEN | 0x0020 | module | 指示模块为开放的 |
| TRANSITIVE | 0x0020 | module requires | 指示依赖于此模块的所有模块都隐式依赖此模块 |
| VOLATILE | 0x0040 | field | volatile字段，在内存中不会缓存 |
| BRIDGE | 0x0040 | method | “桥”方法，由编译器生成 |
| STATIC_PHASE | 0x0040 | module requires | 指示模块在编译时必须但运行时可选 |
| VARARGS | 0x0080 | method | 方法使用@SafeVarargs注释，与static或final连用 |
| TRANSIENT | 0x0080 | field | 被默认序列化忽略 |
| NATIVE | 0x0100 | method | 本地方法（JNI） |
| INTERFACE | 0x0200 | class | 声明类为接口，与abstract连用 |
| ABSTRACT | 0x0400 | class, method | 定义抽象类或抽象方法 |
| STRICT | 0x0800 | method | 严格浮点数定义（strictfp），可能在Java 17失效（？） |
| SYNTHETIC | 0x1000 | class, field, method, parameter, module * | 既不属于显性声明也不属于隐式声明，通常是编译器优化生成 |
| ANNOTATION | 0x2000 | class | 注释类型，与interface、abstract连用 |
| ENUM | 0x4000 | class(?) field inner | 枚举类或枚举字段 |
| MANDATED | 0x8000 | field, method, parameter, module, module * | 隐式声明的数据 |
| MODULE | 0x8000 | class | 声明这个类是模块定义类 |


JVM定义的Access Flags，真正我们能用到的不多，详见<https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.7.25>


ASM自己也有定义Access Flag，由于JVM定义的有效位只有16位，所以这两个标志不会与JVM的访问标志冲突，但是这些标志在写入类之前必须清除（用&操作即可）

| Access Flag | 常量值 | 修饰目标 | 用途 |
| :----:      | :-:   | :-:     | :--- |
| RECORD | 0x10000 | class | 记录类型（record） |
| DEPRECATED | 0x20000 | class, field, method | 弃用，要和@Deprecated连用 |

这些常量可以用or叠加修饰，如果访问标志不合法（比如吧ACC_PUBLIC和ACC_PRIVATE用or联系起来当了访问标志），在ASM写入时是不会报错的，但是在JVM试图加载这个类的时候可能会抛出`ClassFormatError`。

## 解析类的信息：ClassVisitor

ClassVisitor是一个抽象类，它的构造函数仅需要ASM API版本（在`Opcodes`中可以找到，1-9），或者再加上另一个ClassVisitor用于一起解析，下面是一个模板：

```Java
public ClassParser(int api) {
	super(api);
}

public ClassParser() {
	super(ASM9);
}
```

当这个Visitor被传入accept之后，ClassReader会以下面的顺序调用：

```
visit [ visitSource ] [ visitModule ][ visitNestHost ][ visitPermittedSubclass ][ visitOuterClass ] ( visitAnnotation | visitTypeAnnotation | visitAttribute )* ( visitNestMember | visitInnerClass | visitRecordComponent | visitField | visitMethod )* visitEnd
```

不够清晰？那么下面简单说一下流程：

首先访问**类的信息**（`visit`），传入的是**类文件的版本**（version，从V1_1到V16）、**访问标志**（access），**类的全限定名**（name），**泛型签名**（signature，可能为空），**父类全限定名**（无指定为java/lang/Object），**实现接口列表**（全限定名，可为空）

之后访问**注释信息**（`visitAnnotation`），传入的是**注释描述符**（descriptor，这里可能包含有@Repeatable的注释类型，所以这里不是全限定名）和**可见性**（visible，@Retention定义的作用范围，为CLASS传入false，为RUNTIME传入true，为SOURCE不会写入类文件），该方法返回`AnnotationVisitor`。

同时，访问**泛型注释信息**（`visitTypeAnnotation`），传入的是**注释引用类型**（typeRef，可能为TypeReference定义的几个值：CLASS\_TYPE\_PARAMETER \<以泛型类的类型参数为目标的类型引用的类型，常量值0\>，CLASS\_EXTENDS \<以泛型类的超类或它实现的接口之一为目标的类型引用的类型，常量值16\>，CLASS\_TYPE\_PARAMETER\_BOUND \<以泛型类的类型参数的绑定为目标的类型引用的类型，常量值17\>），**泛型类引用路径**（可为空），**注释描述符**和**可见性**，返回AnnotationVisitor。

接着，访问字段、方法和内部类。

字段调用`visitField`方法，传入**访问标志，字段名，描述符，泛型签名和默认值**，返回`FieldVisitor`。

方法调用`visitMethod`方法，传入**访问标志，方法名，描述符，泛型签名和异常列表（全限定名）**，返回`MethodVisitor`。

内部类调用`visitInnerClass`方法，传入**内部类全限定名，外部类全限定名，内部类名称**（不带包路径，也就是没有“.”的名称，如果这个写错了IDE无法识别到这个类，但是不影响调用）**，和访问标志**（这个和类声明定义的标志不同，可以有static，这样类里面就不会带有this$0）。内部类调用指的不只是类中定义了内部类，还包括引用到了其他类的内部类。

当所有信息都访问结束，调用`visitEnd`。

这里的内容只是简单介绍了一下，具体的下文和接下来几篇专栏会写。

## 解析注释信息：AnnotationVisitor

AnnotationVisitor用于解析注释信息，除了最后会调用的`visitEnd`外，其他都与注释类型本身定义的方法返回值有关。下面是不同的类型：

`visit`方法：传入注释方法名称和值，值必须是基本类型（基本数字、char及其数组，String和类）

`visitArray`方法：传入注释方法名称，返回另一个AnnotationVisitor。这个新的Visitor会被传入数组内的值，所有的name传入都为null。注意：visit一个基本数字或char数组等价于使用visitArray，但是在ClassReader解析中不会调用visitArray而是直接调用visit。

`visitAnnotation`方法：传入注释方法名称和值的描述符，返回的是值的AnnotationVisitor。

`visitEnum`方法：传入注释方法名、值的描述符和枚举名称。

对于带有`@Repeatable`注释的注释类型，在Java使用反射时会返回容器注释，也就是在普通编写时有两种等价的编写方式。在ASM中，这两种方式也等价，写入按照第一种处理：

```Java
@T.Ts(value = { @T(value = "ss"), @T(value = "dd") })
public class A { ... }

@T(value = "ss")
@T(value = "dd")
public class A { ... }
```

>对于带有@Repeatable注释的注释类型，这两种使用方式在反射和ASM中完全等价（T.Ts是T的注释容器）


## 解析字段：FieldVisitor

FieldVisitor的构成比较简单，除了`visitEnd`在最后调用外，比较常用的就是`visitAnnotation`和`visitTypeAnnotation`。这些方法的使用都和ClassVisitor的使用差不多，唯一的不同是visitTypeAnnotation的注释引用类型必为`FIELD`（常量值19）

到此简单的解析就讲完了。什么？还差一个MethodVisitor？这是我们之后要说的重要内容，所以这里不会提到它。接下来，是应用ASM的例子。

## 使用范例：解析一个类

解析一个类需要从文章最开始说的ClassReader写起，它能将一个类的字节码解析并且进行Visitor模式调用。在下面的范例中，我们将尝试读取一个类的名称、字段和注释。

首先是一个测试类的编写，之后用javac编译。

```Java
package com.github.nickid2018.asm;

public class TestClass {

	@Deprecated
	public String string;
	public static int integer;
}
```

接着，我们尝试读取这个类的信息，因为测试类和运行ASM的类在同一个项目之下，可以用它的全限定名初始化ClassReader。

```Java
ClassReader reader = new ClassReader("com/github/nickid2018/asm/TestClass");
```

之后我们需要继承三个Visitor：ClassVisitor、FieldVisitor和AnnotationVisitor。我们只需要一些信息，所以不需要将它们的所有方法进行覆盖。

创建一个ClassParser继承ClassVisitor，选择要覆盖的方法。在访问类的时候，我们只需要类名，所以需要覆盖visit；又因为需要解析字段，我们还需要覆盖visitField，并且将我们的字段访问器作为返回值。

```Java
@Override
public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
	super(version, access, name, signature, superName, interfaces);
	System.out.println("类名: " + name);
}

@Override
public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
	System.out.println("字段: " + name + " 描述符: " + descriptor);
	return new FieldParser();
}
```

创建FieldParser继承FieldVisitor解析字段。在读取字段时，我们还需要读取字段中的注释，所以需要覆盖visitAnnotation，返回我们自己的AnnotationVisitor。

```Java
@Override
public AnnotationVisitor visitAnnotation(String descriptor, boolean visible) {
	System.out.println("注释: " + descriptor + " 可见性: " + visible);
	return new AnnotationParser();
}
```

由于@Deprecated不具有任何的注释方法，我们创建的AnnotationParser可以不覆盖任何方法。

这些访问器写完之后，就要递呈给ClassReader开始解析，代码如下：

```Java
ClassParser cv = new ClassParser();
reader.accept(cv, 0);
```

现在，我们的解析程序就完成了。运行结果如下：

```
类名: com/github/nickid2018/asm/TestClass
字段: string 描述符: Ljava/lang/String;
注释: Ljava/lang/Deprecated; 可见性: true
字段: integer 描述符: I
```

代码样例：<https://paste.ubuntu.com/p/8d6jN8jVzr/>

## 使用范例：生成一个类

生成类我们用到的是ClassWriter，它本质上就是ClassVisitor，我们只要用可以构建类的数据按照刚才的格式传给它就能生成对应的类。

它的构造函数有两个，一个只传入一个int，它的值可为三个数：0、COMPUTE_MAXS和COMPUTE_FRAMES。那两个常量值是自动计算方法visitMaxs和visitFrame的，对于现在来说还用不到。另一个构造函数还需要传入ClassReader，这是下一部分可能用到的。

首先确定我们要构建产生的类：

```Java
package com.github.nickid2018.asm;

public class WillGenerate {

	@Deprecated
	public static final int HELLO = 0;
	private String hi;
}
```

首先创建ClassWriter实例：

```Java
ClassWriter cw = new ClassWriter(0);
```

接着，创建类，用到的正是visit方法。由于没有指定父类，这个类的父类将被强行指定为java/lang/Object，接口、抽象类、注释类型也如此。这个类没有实现任何接口，所以interfaces可以传null。同理，它没有泛型，所以泛型的signature为null。访问标志是public，再加上super，整体下来就是这句：

```Java
cw.visit(V1_8, ACC_PUBLIC + ACC_SUPER, "com/github/nickid2018/asm/WillGenerate", null, "java/lang/Object",null);
```

接下来我们需要创建默认构造函数。javac编译时会把没有定义构造函数的普通类加入默认的构造函数。这种构造函数里面包括了父类构造函数调用和本身的非基本类型字段赋值。如果没有非基本类型字段赋值，那么它的代码就像这样：

```Java
public WillGenerate() {
	super();
}
```

由于这篇专栏主要是有关于类、字段、注释的解析，方法的解析暂时先不讲，所以这里只给出它的写入代码，不做讲解。

```Java
public static void writeDefaultInit(ClassWriter cw) {
	MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
	mv.visitVarInsn(ALOAD, 0);
	mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
	mv.visitInsn(RETURN);
	mv.visitMaxs(1, 1);
	mv.visitEnd();
}
```

接下来写入HELLO这个字段。它的访问标志是public+static+final，由于它是弃用的，它也可以加上deprecated这个ASM自己定义的Access Flag。它的类型是int，所以描述符是I。没有泛型，所以signature为null。有默认值，为0。所以它的写入像这样：

```Java
FieldVisitor fv = cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC + ACC_DEPRECATED, "HELLO", "I", null, (Integer) 0);
```

保留这个FieldVisitor，因为它还具有一个注释@Deprecated。注释类型的描述符为Ljava/lang/Deprecated;。又因为@Deprecated的作用范围是RUNTIME，所以可见性为true，代码如下：

```Java
AnnotationVisitor av = fv.visitAnnotation("Ljava/lang/Deprecated;", true);
```

这时，这个字段就写入信息就完成了，调用visitEnd。

```Java
av.visitEnd();
fv.visitEnd();
```

下面写hi这个字段，和上面的差不多，直接给代码：

```Java
fv = cw.visitField(ACC_PRIVATE, "hi", "Ljava/lang/String;", null, null);
fv.visitEnd();
```

这时候类的所有信息都已经写完了，调用ClassWriter的visitEnd。

```Java
cw.visitEnd();
```

接下来调用ClassWriter的toByteArray获得字节码信息，写入到文件中就能得到类。

运行之后调用反编译器的结果：

```Java
package com.github.nickid2018.asm;

public class WillGenerate
{

	@Deprecated
	public static final int HELLO = 0;
	private String hi;
}
```

代码样例：https://paste.ubuntu.com/p/cqfDPVZbsH/

## 使用范例：修改一个类

修改类需要ClassReader和ClassWriter互相配合。利用ClassVisitor等进行数据的转移和修改。

接下来用ASM改一下我们的TestClass。

```Java
public class TestClass {				// 改为抽象类

	@Deprecated
	public String string;               // 重命名为str
	public static int integer;			// 加上final和默认值10
}
```

首先，创建ClassReader和ClassWriter。

```Java
ClassWriter cw = new ClassWriter(0);
ClassReader cr  = new ClassReader("com/github/nickid2018/asm/TestClass");
```

之后在我们的ClassParser里面改一下，传入一个ClassWriter，使用父类的第二个构造函数：以int，ClassVisitor为参数的构造函数。这样，ClassReader传入的信息可以直接写到ClassWriter里面，我们只需要修改我们所需要的方法就可以达到修改的效果，而不用将所有ClassVisitor的方法实现。

```Java
public ClassParser(ClassWriter cw) {
	super(ASM9, cw);
}
```

接下来解决第一个修改：改为抽象类。这个我们可以在visit里面修改，将原先的访问标志加一个abstract就好。

```Java
@Override
public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
	super.visit(version, access + ACC_ABSTRACT, name, signature, superName, interfaces);
}
```

第二个修改是重命名字段。这个在visitField里面判断就行，像下面一样：

```Java
if (name.equals("string"))
	return super.visitField(access, "str", descriptor, signature, value);
```

第三个就是修改为final和加默认值，也是在visitField里面改动：

```Java
if (name.equals("integer"))
	return super.visitField(access + ACC_FINAL, name, descriptor, signature, (Integer) 10);
```

最后用accept传入ClassParser，输出文件就是改好的类文件。

```Java
cr.accept(new ClassParser(cw), 0);
```

生成之后，用反编译器看一下结果。

```Java
package com.github.nickid2018.asm;

public abstract class TestClass
{

	@Deprecated
	public String str;
	public static final int integer = 10;
}
```

代码样例：<https://paste.ubuntu.com/p/yXVvdJs3WH/>

***

这篇专栏到这里就结束了，下一期专栏：MethodVisitor和Opcode（一）

如果文章中有任何错误，可以在评论区留言，我将会修正错误。

如果使用ASM中有问题，可以在下面评论。