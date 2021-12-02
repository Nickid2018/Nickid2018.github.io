---
title: Java ASM详解：MethodVisitor和Opcode（二）类型、数组、字段、方法、异常与同步
date: 2021-10-05 12:22:46
category:
  - Java
tags:
  - Java
  - ASM
toc: true
---
上次讲过了操作栈与数值运算操作，这篇专栏主要讲ASM中有关于类型、数组与方法调用的字节码。

*P.S.ASM库已经更新到了9.2版本，可以试试解析Java 18的类了。*

## 有关于类型的字节码

有关于类型的字节码都是用visitTypeInsn进行写入的。这类字节码共有4个：NEW，ANEWARRAY，INSTANCEOF和CHECKCAST。ANEWARRAY在之后的数组字节码里面会仔细去讲。

### new

```Java
输入：无
输出：一个指定类型的对象
方法：visitTypeInsn
参数：类型
使用范例：
	mv.visitTypeInsn(NEW, "java/lang/Object");
```

NEW只进行创建对象，不负责调用构造函数，所以内部字段的值都为默认值。调用构造函数必须用invokespecial字节码进行调用（下文）。

在调用这个字节码时，如果指向的类没有初始化，就它的调用静态初始化函数&lt;clinit&gt;。如果在初始化中发生异常就会抛出错误。如果目标类的类格式有误，则抛出异常。如果目标类时抽象的，则抛出`InstantiationError`。

### instanceof

```Java
输入：一个对象
输出：布尔值，代表是否为指定类的对象（栈上表示为一个四字节数据）
方法：visitTypeInsn
参数：类型（对于数组是描述符）
使用范例：
  mv.visitTypeInsn(INSTANCEOF, "java/lang/String");
```

instanceof用于检查对象是否为这个类型的实例，如果是则返回boolean值true，即操作栈上的一个int数据1；如果不是就返回0。

对于null对象，该字节码永远返回0。

### checkcast

```Java
输入：一个对象
输出：类型检查后的对象
方法：visitTypeInsn
参数：类型（对于数组是描述符）
使用范例：
  mv.visitTypeInsn(CHECKCAST, "java/io/InputStream");
```

checkcast用于检查对象的类型，类似于instanceof。但不同的是，如果无法将对象转换为指定类型，该字节码会抛出ClassCastException。这个字节码经常见于泛型中。

加入这个字节码通常是为了指定对象是某个类型好让验证器验证，在局部变量无法得知确切类型时必须加入此字节码保证验证通过（运行时报错就是另一回事了）。


下面是这三个字节码组合的例子：

要生成的Java代码如下：

```Java
public static String testTypeASM() {
  Object object = new String("Hello");
  boolean bool = object instanceof String;
  return (String) object;
}
```

对应的生成这段代码的字节码程序如下：

```Java
// 省略了ClassWriter和MethodVisitor的创建，mv是MethodVisitor实例
// ---- 第一行语句 -----
mv.visitTypeInsn(NEW, "java/lang/String");
mv.visitInsn(DUP); // 栈上复制一份对象
mv.visitLdcInsn("Hello");
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/String", "<init>", "(Ljava/lang/String;)V", false); // 执行构造函数
mv.visitVarInsn(ASTORE, 0); // 存在局部变量表0号位
// ---- 第二行语句 -----
mv.visitVarInsn(ALOAD, 0); // 取出局部变量object
mv.visitTypeInsn(INSTANCEOF, "java/lang/String");
mv.visitVarInsn(ISTORE, 1);
// ---- 第三行语句 -----
mv.visitVarInsn(ALOAD, 0);
mv.visitTypeInsn(CHECKCAST, "java/lang/String"); // 注：此处可以不加这个CHECKCAST，因为局部变量表已知是String。如果局部变量表无法判断是否真的为String且没有加入这个语句，在验证时下方ARETURN字节码会报错
mv.visitInsn(ARETURN);
// ---- 结束 ----
mv.visitMaxs(3, 2);
mv.visitEnd();
```

## 数组操作的字节码

数组操作的字节码一共有20个，其中加载指令8个，存储指令8个，三个创建还有一个获取数组长度的字节码。

### newarray

```Java
输入：int，代表数组长度
输出：指定长度的数组
方法：visitIntInsn
参数：数组的类型，有8个常量值分别代表了不同的值类型
使用范例：
  mv.visitIntInsn(NEWARRAY, T_INT);
```

和newarray字节码用于创建基本类型的数组，它的参数代表了它的类型，在Opcodes类中一共有8个：T_BOOLEAN（boolean），T_CHAR（char），T_FLOAT（float），T_DOUBLE（double），T_BYTE（byte），T_SHORT（short），T_INT（int）和T_LONG（long）。

如果数组长度小于0，这个字节码会抛出`NegativeArraySizeException`。

### anewarray

```Java
输入：int，代表数组长度
输出：指定长度的数组
方法：visitTypeInsn
参数：类型
使用范例：
  mv.visitTypeInsn(ANEWARRAY, "java/lang/String");
```

基本类型的数组由newarray创建，而不是基本类型的数组由anewarray创建。

和newarray一样，如果数组长度小于0，这个字节码会抛出`NegativeArraySizeException`。

### multianewarray

```Java
输入：一系列的int，代表多维数组中每一维的长度
输出：多维数组
方法：visitMultiANewArrayInsn
参数：描述符和维度
使用范例：
  mv.visitMultiANewArrayInsn("[[Ljava/lang/String;", 2);
  mv.visitMultiANewArrayInsn("[[[I", 3);
```

创建一个多维数组，多维数组的描述符要与第二个参数维度相匹配。和另两个字节码相同，如果多维数组任意一维的长度小于0，这个字节码就会抛出`NegativeArraySizeException`。

下面是使用这三个字节码的例子：

Java代码：

```Java
int[] arrayInt = new int[10];
int[][] multi2Int = new int[100][2];
String[] strings = new String[30];
String[][] multiStrings = new String[127][128];
```

生成这些代码的字节码程序：

```Java
// 省略了ClassWriter和MethodVisitor的创建，mv是MethodVisitor实例
// new int[10]
mv.visitIntInsn(BIPUSH, 10);
mv.visitIntInsn(NEWARRAY, T_INT);
mv.visitVarInsn(ASTORE, 0);
// new int[100][2]
mv.visitIntInsn(BIPUSH, 100);
mv.visitInsn(ICONST_2);
mv.visitMultiANewArrayInsn("[[I", 2);
mv.visitVarInsn(ASTORE, 1);
// new String[30]
mv.visitIntInsn(BIPUSH, 30);
mv.visitTypeInsn(ANEWARRAY, "java/lang/String");
mv.visitVarInsn(ASTORE, 2);
// new String[127][128]
mv.visitIntInsn(BIPUSH, 127);
mv.visitIntInsn(SIPUSH, 128);
mv.visitMultiANewArrayInsn("[[Ljava/lang/String;", 2);
mv.visitVarInsn(ASTORE, 3);
```

在创建数组时，如果是一维数组就用newarray或anewarray。multianewarray也能创建一维数组，但是使用上面的两个更加高效。

### arraylength

```Java
输入：数组
输出：数组长度
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(ARRAYLENGTH);
```

获取数组的长度，返回int。如果数组输入为null，抛出空指针异常。

### *x*aload

x=a,b,c,d,f,i,l,s, 其中b同时负责了byte和boolean

```Java
输入：数组，int类型的下标
输出：数组元素
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(BALOAD);
```

xaload的作用是从数组指定下标取元素。如果下标超过数组长度，抛出`ArrayIndexOutOfBoundsException`。对于多维数组的提取元素方式类似下面：

```Java
// Java 代码：
// 设int[][] multi = new int[10][20];位于局部变量表0位
return multi[1][4];
// 字节码：
// 省略了ClassWriter和MethodVisitor的创建，mv是MethodVisitor实例
mv.visitVarInsn(ALOAD, 0);
mv.visitInsn(ICONST_1);
mv.visitInsn(AALOAD);
mv.visitInsn(ICONST_4);
mv.visitInsn(IALOAD);
mv.visitInsn(IRETURN);
```

### *x*astore

x=a,b,c,d,f,i,l,s, 其中b同时负责了byte和boolean

```Java
输入：数组，int类型的下标，一个变量
输出：无
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(BASTORE);
```

将对象存入数组指定下标。如果下标超过数组长度，抛出`ArrayIndexOutOfBoundsException`。对于多维数组，存储对象需要和xaload一起配合。

```Java
// Java 代码：
// 设int[][] multi = new int[10][20];位于局部变量表0位
multi[1][4] = 20;
// 字节码：
// 省略了ClassWriter和MethodVisitor的创建，mv是MethodVisitor实例
mv.visitVarInsn(ALOAD, 0);
mv.visitInsn(ICONST_1);
mv.visitInsn(AALOAD);
mv.visitInsn(ICONST_4);
mv.visitIntInsn(BIPUSH, 20);
mv.visitInsn(IASTORE);
```

## 操作字段的字节码

在代码中我们经常会调用类中的字段，例如`System.out`。Java提供了四个字节码用于访问和修改字段。

### getfield

```Java
输入：一个对象
输出：对应字段的值
方法：visitFieldInsn
参数：字段所处的类、字段名、字段描述符
使用范例：
	mv.visitFieldInsn(GETFIELD, "org/objectweb/asm/MethodVisitor", "mv", "Lorg/objectweb/asm/MethodVisitor;");
```

getfield用于获取非静态字段的值。如果它作用目标是一个静态字段，则在类连接验证时抛出`IncompatibleClassChangeError`。

如果输入的对象是null，这个字节码会在运行时抛出空指针异常。

这个字节码不能调用数组的length字段，在编译的时候length字段会自行转变成arraylength字节码。

### getstatic

```Java
输入：无
输出：对应字段的值
方法：visitFieldInsn
参数：字段所处的类、字段名、字段描述符
使用范例：
 mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
```

getstatic用于获取静态字段的值。如果它作用目标是一个非静态字段，则在类连接验证时抛出`IncompatibleClassChangeError`。

### putfield

```Java
输入：一个对象、准备修改成的对象
输出：无
方法：visitFieldInsn
参数：字段所处的类、字段名、字段描述符
使用范例：
 mv.visitFieldInsn(PUTFIELD, "org/objectweb/asm/MethodVisitor", "mv", "Lorg/objectweb/asm/MethodVisitor;");
```

putfield用于修改非静态字段的值。如果它作用目标是一个静态字段，则在类连接验证时抛出`IncompatibleClassChangeError`。

如果输入的对象是null，这个字节码会在运行时抛出空指针异常。

对于final字段，如果不是在初始化对象时修改（构造函数中），那么就会抛出`IllegalAccessError`。

### putstatic

```Java
输入：准备修改成的对象
输出：无
方法：visitFieldInsn
参数：字段所处的类、字段名、字段描述符
使用范例：
 mv.visitFieldInsn(PUTSTATIC, "io/github/nickid2018/Constants", "test", "Z");
```

putstatic用于修改静态字段的值。如果它作用目标是一个非静态字段，则在类连接验证时抛出`IncompatibleClassChangeError`。

对于final字段，如果不是在类初始化时修改（&lt;clinit&gt;中），那么就会抛出`IllegalAccessError`。

## 调用方法的字节码

调用方法的字节码共有五个：invokevirtual，invokespecial，invokestatic，invokeinterface和invokedynamic。invokedynamic使用了BSM（BootStrap Method），讲解起来很复杂，所以这个要单独分出来一篇文章去讲。这篇文章主要讨论前四个。

### invokevirtual

```Java
输入：一个对象，传入参数
输出：与方法返回值有关
方法：visitMethodInsn
参数：方法所在的类，方法名，方法描述符，固定值false
使用范例：
 mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
```

这个字节码用于调用实例方法：如果对象是子类的对象且子类复写了这个方法，则调用子类的方法；如果对象就是该类的直接对象或者对象所属子类没有复写这个方法，就调用现在类的方法。

在编译时，如果子类调用了父类的方法且子类没有实现此方法，那么方法所在的类要写为父类。如果使用`super`，要用invokespecial调用（下文）。

如果方法调用目标是静态的，在连接验证时会抛出`IncompatibleClassChangeError`。

如果方法调用目标是抽象的，并且在继承树上没有任何实现此方法的类，在调用时会抛出`AbstractMethodError`。

如果方法调用目标是抽象的，而继承树上由多个实现此方法的类，且这些方法都是可被选中成为调用目标的方法（比如一个类继承于一个抽象类，又实现了两个接口，两个接口中都有一个同样的default方法可作为抽象类中抽象方法的实现目标），这时此字节码会抛出`IncompatibleClassChangeError`。

如果方法调用目标是native的，且没有任何JNI连接查询到这个方法和哪个C函数相连接，这时这个字节码抛出`UnsatisfiedLinkError`。

### invokespecial

```Java
输入：一个对象，传入参数
输出：与方法返回值有关
方法：visitMethodInsn
参数：方法所在的类，方法名，方法描述符，固定值false
使用范例：
 mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "equals", "(Ljava/lang/Object;)Z", false);
```

invokespecial类似于invokevirtual，但不同的是，它和调用方法的对象的类型无关：它的方法调用对象就是字节码内部标定的方法，如果这个类找不到就寻找直接超类的方法，而不是像invokevirtual要考虑继承树所有的方法。

这个方法经常在构造函数中看到，因为无论什么类都需要有一个构造函数，而构造函数内部必须自动调用父类构造函数。

一个默认的构造函数类似于下面：

```Java
public class Test {

  public Test() {
    super();
  }
}
```

在生成类时，如果没有自定义其他构造函数，就要加上这个默认构造函数：

```Java
ClassWriter cw = new ClassWriter(0);
cw.visit(V17, ACC_PUBLIC + ACC_SUPER, "Test", null, "java/lang/Object", null);
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
mv.visitVarInsn(ALOAD, 0); // 加载自身(this)
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false); // 调用父类构造函数
mv.visitInsn(RETURN);
mv.visitMaxs(1, 1);
mv.visitEnd();
cw.visitEnd();
```

### invokestatic

```Java
输入：一个对象，传入参数
输出：与方法返回值有关
方法：visitMethodInsn
参数：方法所在的类，方法名，方法描述符，固定值false
使用范例：
 mv.visitMethodInsn(INVOKESTATIC, "java/lang/Math", "sin", "(D)D", false);
```

invokestatic用于调用静态方法，如果调用目标不是个静态方法，抛出`IncompatibleClassChangeError`。

和invokevirtual一样，如果目标是个native方法而JNI找不到连接的C函数，该字节码抛出`UnsatisfiedLinkError`。

### invokeinterface

```Java
输入：一个对象，传入参数
输出：与方法返回值有关
方法：visitMethodInsn
参数：方法所在的类，方法名，方法描述符，固定值true
使用范例：
 mv.visitMethodInsn(INVOKEINTERFACE, "java/util/Set", "clear", "()V", true);
```

这个字节码类似于invokevirtual，异常情况的处理也和它类似。它用于调用接口方法，而不是像invokevirtual的实例方法。

## 抛出异常的字节码：athrow

```Java
输入：一个Throwable对象
输出：操作栈不变
方法：visitInsn
参数：无
使用范例：
 mv.visitInsn(ATHROW);
```

athrow负责将一个Throwable对象抛出。如果对象是null，那么就不会抛出这个null，而是抛出`NullPointerException`。

通常情况下，我们都是直接new一个Throwable对象然后直接抛出，就像这样：

```Java
throw new Exception("error!");
```

翻译为字节码如下：

```Java
// 省略了ClassWriter和MethodVisitor的创建，mv是MethodVisitor实例
mv.visitTypeInsn(NEW, "java/lang/Exception");
mv.visitInsn(DUP);
mv.visitLdcInsn("error!");
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Exception", "<init>", "(Ljava/lang/String;)V", false);
mv.visitInsn(ATHROW);
```

## 同步字节码

同步操作共有两个字节码，monitorenter和monitorexit，成套使用。

```Java
输入：一个对象
输出：无
方法：visitInsn
参数：无
使用范例：
 mv.visitInsn(MONITORENTER);
 mv.visitInsn(MONITOREXIT);
```

输入的对象必须是引用类型对象，不能是基本类型的值。

使用同步块时，代码类似这样：

```Java
Object lock = new byte[0];//设0号位
synchronized(lock) {
  //...
}
```

对应的字节码：

```Java
// 省略了ClassWriter和MethodVisitor的创建，mv是MethodVisitor实例
mv.visitInsn(ICONST_0);
mv.visitIntInsn(NEWARRAY, T_BYTE);
mv.visitVarInsn(ASTORE, 0);
mv.visitVarInsn(ALOAD, 0);
mv.visitInsn(MONITORENTER);
//...
mv.visitVarInsn(ALOAD, 0);
mv.visitInsn(MONITOREXIT);
```

monitorenter就是尝试加锁的操作。如果这个对象的监视器条目计数为0，此线程会把这个计数设置为1，这时此线程就是这个对象的监视器；如果不为0且线程不是该对象的监视器，线程会阻塞直到计数为0时重新尝试加锁；如果线程已经是这个对象的监视器，计数递增。

monitorexit就是释放锁的操作。如果线程是这个对象的监视器，计数递减，当计数减为0时该线程就不是这个对象的监视器了。如果线程不是这个对象的监视器，这个字节码会抛出`IllegalMonitorStateException`。

monitorenter可以和很多monitorexit一起出现，在一个方法的所有可能流程中的加锁次数和释放次数必须相同，否则在调用时会发生`IllegalMonitorStateException`。

对于同步方法（访问标志含有ACC_SYNCHRONIZED），不需要手动对自身对象或类加锁。JVM在调用方法前隐式加锁，在调用之后隐式释放。

## 应用：计算两数之积

学到了这些字节码，接下来我们要试试用纯字节码解决这道简单的问题。

```
输入：两个双精度浮点数a,b
输出：一个保留5位小数部分的双精度浮点数，代表a*b

例：
3.22 6.11
输出：
19.67420
```

在Java代码下，我们可以这样写：

```Java
// 这里不写main方法，而是写了一个静态的test方法用于后续调用
import java.util.Scanner;

public class Test {

  public static void test() {
    Scanner scanner = new Scanner(System.in);
    double a = scanner.nextDouble();
    double b = scanner.nextDouble();
    System.out.printf("%.5f", a * b);
  }
}
```

下面是用ASM生成的步骤：

首先还是创建类和方法，不再多说。

第一行，创建Scanner对象，这里用到的就是new。

```Java
// 省略了ClassWriter和MethodVisitor的创建，mv是MethodVisitor实例
mv.visitTypeInsn(NEW, "java/util/Scanner");
mv.visitInsn(DUP);
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "in", "Ljava/io/InputStream;");
mv.visitMethodInsn(INVOKESPECIAL, "java/util/Scanner", "<init>", "(Ljava/io/InputStream;)V", false);
mv.visitVarInsn(ASTORE, 0);
```

第二行和第三行都是读取double，这里是调用了Scanner的`nextDouble`方法，这里只给第二行的例子：

```Java
mv.visitVarInsn(ALOAD, 0);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/util/Scanner", "nextDouble", "()D", false);
mv.visitVarInsn(DSTORE, 1);
```

接下来是个重头戏。首先来看看`PrintStream::printf`的定义：

```Java
public PrintStream printf(String format, Object ... args)
```

可以看到，args是个不定长参数，这怎么表示呢？

在Java中，不定长参数都被解析为数组，也就是说，它在字节码中的表示其实是这样的：

```Java
public PrintStream printf(String format, Object[] args)
```

现在我们需要传递的参数就是一个字符串和一个Object数组。可是double不是引用类型，这又要怎么办呢？

在Java中，基本类型都有它们的“包装类”。double的包装类是java.lang.Double，通过`Double::valueOf`方法就可以把double值转变为Double对象，也就是装箱操作。在平常编写时，Java编译器会自动为我们添加装箱操作，也就是自动装箱。

经过这样的解析，最后这句话的Java代码表示就像这样：

```Java
System.out.printf("%.5f", new Object[] { Double.valueOf(a * b) });
```

其中Object[]是一个长度为1的数组，也就是先创建它然后将Double对象用aastore字节码放入就行。

```Java
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"); // System.out
mv.visitLdcInsn("%.5f"); // printf的第一个参数
mv.visitInsn(ICONST_1); // Object[]的长度
mv.visitTypeInsn(ANEWARRAY, "java/lang/Object"); // 创建Object[]
mv.visitInsn(DUP); // 复制一份数组，一份用于放入对象，一份用于传入方法
mv.visitInsn(ICONST_0); // 放入数组的位置，0
mv.visitVarInsn(DLOAD, 1); // 取出a
mv.visitVarInsn(DLOAD, 3); // 取出b（3是因为double要占两个局部变量槽位！）
mv.visitInsn(DMUL); // 计算 a * b
mv.visitMethodInsn(INVOKESTATIC, "java/lang/Double", "valueOf", "(D)Ljava/lang/Double;", false); // 装箱
mv.visitInsn(AASTORE); // 放入数组
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "printf", "(Ljava/lang/String;[Ljava/lang/Object;)Ljava/io/PrintStream;", false);
```

最后写入return和visitMaxs，局部变量一共5个槽位，最大的操作栈大小是9：

```Java
mv.visitInsn(RETURN);
mv.visitMaxs(9, 5);
```

下面就可以实验了！

```
3.22 6.11
19.67420
```

测试结果和预测一样！

全部代码：<https://paste.ubuntu.com/p/NXDfFpQ4y6/>

***

这篇专栏的内容结束了，下一篇：Java ASM详解：MethodVisitor与Opcode（三）标签，选择结构，循环结构，栈帧

这篇文章一共讲了34个字节码，从开始到现在一共讲了164个。

有错误在评论中指出。