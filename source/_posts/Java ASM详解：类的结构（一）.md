---
title: Java ASM详解：类的结构（一）
layout: default
date: 2022/3/30
category:
	- Java
tags:
 - Java
 - ASM
comments: true
toc: true
---

了解了各个字节码的意义，我们可以构建出方法。这篇文章开始不再讲具体的字节码，而是开始介绍类的结构。今天这篇文章主要讲类的成员/属性和它们在字节码中的写入表示。

## 类静态初始化方法

当一个类被装载入内存，它是没有`静态初始化`过的。当从其他类调用它内部的方法或字段时，类才会被静态初始化。静态初始化只进行一次。

静态初始化的主要工作是在类加载之后使用之前进行内部数据的初始化操作。在Java代码中，它使用`static块`声明，一个类文件可以有多个static块。

```Java
static {
  // Code...
}
```

在字节码中，静态初始化会被写入成为一个方法，名称为`<clinit>`，是`Class Initialization`的缩写。它的描述符要求是`()V`，不带有泛型签名，不抛出异常，访问标志必须含有`static`。如果&lt;clinit&gt;不满足这些条件，会产生下面这些报错：

| 错误输入 | 报错 |
| :-: | :-- |
| 访问标志不存在static | java.lang.ClassFormatError: Method &lt;clinit&gt; is not static in class file \* |
| 方法描述符不是`()V` | java.lang.ClassFormatError: Method "&lt;clinit&gt;" in class \* has illegal signature \* |

和static块不同，在字节码中&lt;clinit&gt;**只能存在一个**。多个static块合成一个&lt;clinit&gt;会按照static块的顺序一块一块进行拼接，同时局部变量也会进行拼接。例如下方的代码：

```Java
static {
  String a = "a";
  System.out.println(a);
}

static {
  Object a = "b";
  System.out.println(a);
}
```

在字节码中会进行拼接，翻译后变成这样：

```Java
static {
  {
    String a = "a";
    System.out.println(a);
  }
  {
    Object a = "b";
    System.out.println(a);
  }
}
```

## 构造函数

创建某个类的对象必然会调用某一个具体的构造函数。构造函数的意义就是对类对象内部数据进行初始化。

在字节码中，构造函数的名称是`<init>`而非类名。它要求返回值是`void（V）`，访问标志只包含访问权限标志（public/protected/private），对于参数列表和异常列表不加限制。如果不满足上面的条件，JVM在加载阶段会抛出下面的异常：

| 错误输入 | 报错 |
| :-: | :-- |
| 访问标志存在不合法的标志 | java.lang.ClassFormatError: Method &lt;init&gt; in class \* has illegal modifiers: \* |
| 方法描述符返回值不是`V` | java.lang.ClassFormatError: Method "&lt;init&gt;" in class \* has illegal signature \* |

构造函数的另一项限制在它的内部。构造函数必须调用它父类的构造函数或本类的另一个构造函数，否则会在验证时抛出`java.lang.VerifyError: Constructor must call super() or this() before return`。

每个类都要含有一个构造函数。如果源码中没有构造函数，那么编译时会自动添加默认构造函数，它的Java源码和字节码写入如下：

【注意：这里的代码不适用于非静态内部类，下文会详细介绍】

```Java
public ClassName() {
  super();
}

// --- 字节码
mv.visitVarInsn(ALOAD, 0); // 加载this
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false); // 类名需要改成父类
mv.visitInsn(RETURN);
mv.visitMaxs(1, 1);
```

## 类字段的初始化

类中的静态字段不和类对象绑定而和类绑定，因此它们必须在静态初始化时被初始化；而类中的实例字段和对象绑定，需要在对象构造时被初始化。

* 如果初始化语句是一个常量字面值且字段是静态字段，如`字符串`、`数字`、`null`或`XXXX.class`，它们的初始化字节码应该直接使用`visitField`的`value`参数将初始值传入。
* 如果初始化语句不是常量字面值或不是静态字段，如`调用方法语句`、`对象创建`、`数组创建`等，它们的初始化字节码应该在`<clinit>`或`<init>`方法内。

根据这个规则，我们可以推断出一段静态字段初始化的代码的具体实现：

```Java
public static final int CONST_INT = 1;
public static final long START_TIME = System.currentTimeMillis();
public static Object object;
```

在写入字节码时，Java代码应该是这样的：

```Java
public static final int CONST_INT = 1;
public static final long START_TIME;
public static Object object;

static {
  START_TIME = System.currentTimeMillis();
}
```

写入字节码的代码：

```Java
// 类名为Test，省略ClassWriter创建
FieldVisitor fv = cw.visitField(ACC_PUBLIC + ACC_STATIC + ACC_FINAL, "CONST_INT", "I", null, 1);
fv.visitEnd();
fv = cw.visitField(ACC_PUBLIC + ACC_STATIC + ACC_FINAL, "START_TIME", "J", null, null);
fv.visitEnd();
fv = cw.visitField(ACC_PUBLIC + ACC_STATIC, "object", "Ljava/lang/Object;", null, null);
fv.visitEnd();
MethodVisitor mv = cw.visitMethod(ACC_STATIC, "<clinit>", "()V", null, null);
mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
mv.visitFieldInsn(PUTSTATIC, "Test", "START_TIME", "J");
mv.visitInsn(RETURN);
mv.visitMaxs(2, 0);
mv.visitEnd();
```

在字节码中，静态初始化方法内可以对一个静态常量字段进行多次赋值，并且JVM不报错。如果在静态初始化中不存在初始化某个静态字段的代码，那么它们就会使用默认值，也就是`visitField`中`value`参数决定的值。

实例字段的初始化类似于静态初始化，只是它们在构造函数内写入。

## 桥接方法

在介绍桥接方法（Bridge Method）之前，先来简单介绍**重写**（Override）。

重写就是子类将父类的某个方法进行覆盖，进而实际执行时会执行子类方法而不是父类的。重写需要满足：

* **名称相同**：父类的方法名称必须与子类的方法名称相同。
* **参数列表对应**：父类的方法参数列表应该与子类一一对应，这一点适用于泛型，也就是子类确定的类型参数应该在重写方法中带入类型参数确定的类型。
* **异常列表不增添**：子类的复写方法不能出现父类没有声明抛出的异常。
* **访问权限不缩小**：子类的复写方法的可见性不能低于父类方法可见性，如父类的访问可见性为`public`，那么子类也必须声明为`public`。

下面是具体的例子，下方的重写案例都是正常能通过编译的：

```Java
interface SupClass<T, R> {
  void test1();
  void test2(T t);
  R test3();
  R test4(T t);
}

class BaseClass<R> implements SupClass<String, R> {
  public void test1() ...
  public void test2(String s) ...
  public R test3() ...
  public R test4(String s) ...
}
```

之前我们说到，泛型的实现是所谓的**泛型擦除**，也就是类型参数会被擦除到其限定的父类上。现在来看看`test2`这个方法，在父类和子类中，它们的方法描述符和泛型签名是不一样的：

```Java
SupClass test2:  (Ljava/lang/Object;)V <T:Ljava/lang/Object;>(T)V
BaseClass test2: (Ljava/lang/String;)V null
```

虽然描述符不同，但是在逻辑上已经达成了重写条件，应该当作重写处理。但是，因为方法描述符不同，实际调用时JVM是找不到这个方法的：`invokevirtual`字节码**只会寻找名称相同且方法描述符相同的方法**。因此，桥接方法出现用于解决这个问题。它的代码意义就是将确定的类型参数强制转换，将父类泛型化的参数传入具体化的子类复写方法中。

例如test2，编译器给出的桥接方法就像下面这样：

```Java
// BaseClass
// 注意：不要在源码中这样写！编译器会因为"具有相同擦除但不构成重写"抛出异常
public void test2(Object s) { // 父类的方法描述符
  test2((String) s); // 通过强制类型转换，将参数列表转换以满足子类的方法描述符
}
```

桥接方法仅出现在**父类方法和子类重写方法擦除后的方法描述符不一致**时，如下方的例子：

```Java
interface SupClass<T, R, E extends Exception> {
  void test1(T t);
  R test2();
  void test3() throws E;
}

class BaseClass<R extends InputStream> implements SupClass<String, R, IOException> {
  void test1(String s) ...
  // 需要生成桥接方法：参数列表描述符不同
  R test2() ...
  // 需要生成桥接方法：R擦除到的类型和父类擦除不一样
  void test3() throws IOException ...
  // 不需要生成桥接方法：方法描述符相同
}
```

桥接方法拥有下面的特性：

* 名称与方法描述符相同：为了`invokevirtual`字节码能成功定位到这个重写方法，桥接方法必须和父类的目标方法名称和描述符一致。
* 访问标志带有`ACC_SYNTHETIC`和`ACC_BRIDGE`标志。
* 访问权限和子类重写方法相同。
* 异常列表和父类方法相同。

接下来我们要使用字节码实现`BaseClass`：

```Java
interface SupClass<T> {
  void test(T t):
}

class BaseClass implements SupClass<String> {
  
  public void test(String s) {
    System.out.println(s);
  }
}
```

字节码：

```Java
// 略过构造函数，cw是ClassWriter
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "test", "(Ljava/lang/String;)V", null, null);
mv.visitVarInsn(ALOAD, 1);
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitInsn(RETURN);
mv.visitMaxs(2, 2);
mv.visitEnd();
// 桥接方法
mv = cw.visitMethod(ACC_PUBLIC + ACC_SYNTHETIC + ACC_BRIDGE, "test", "(Ljava/lang/Object;)V", null, null);
mv.visitVarInsn(ALOAD, 0);
mv.visitVarInsn(ALOAD, 1);
mv.visitTypeInsn(CHECKCAST, "java/lang/String");
mv.visitMethodInsn(INVOKEVIRTUAL, "Test", "test", "(Ljava/lang/String;)V", false);
mv.visitInsn(RETURN);
mv.visitMaxs(2, 2);
mv.visitEnd();
```

## 内部类

类的内部成员除了字段和方法外，还有内部类。内部类分为两种：

* 静态内部类：使用static修饰的内部类。内部接口、内部枚举默认带有static访问标志，因此它们也属于静态内部类。
* 非静态内部类：不使用static修饰的内部类。

内部类的字节码数据不应该在外部类中的类文件数据中出现，应该独立于外部类。命名方式为`外部类名称$内部类名称`；如果内部类是局部内部类，在内部类名称前还需要加上编号；如果内部类是一个匿名内部类，应该使用编号代替。

虽然内部类不需要写在外部类文件里面，但是外部类文件还是要声明它的。声明使用`ClassWriter`的`visitInnerClass`方法，它的每个参数的意义如下：

* name - 内部类的全限定名。例如`test.Test`下的内部类`Inner`这项值就是`test/Test$Inner`。
* outerName - 外部类的全限定名，如果内部类是匿名内部类或局部内部类，这项是`null`。
* innerName - 内部类的名称，如果内部类是匿名内部类，这项是`null`。
* access - 内部类的访问标志。

最外层外部类需要写出它内部所有的类，包括嵌套的内部类。

下面是一些内部类和它们的声明：

```Java
class Test {
  private class Inner1 { ... } // 内部类1 - 成员内部类
  public void test() {
    class Inner2 { ... } // 内部类2 - 局部内部类
    System.out.println(new Test() { ... }); // 内部类3 - 匿名内部类
  }
}

// --- 字节码 外部类Test cw是ClassWriter
cw.visitInnerClass("Test$Inner1", "Test", "Inner1", ACC_PRIVATE);
cw.visitInnerClass("Test$1Inner2", null, "Inner2", 0); // 局部内部类无访问标志
cw.visitInnerClass("Test$1", null, null, 0); // 匿名内部类无访问标志
```

在Java 11，JEP 181（Nest-Based Access Control）加入了`NestHost`和`NestMember`两项属性用于辅助访问权限控制，规定了所有内部类（包括嵌套的内部类）是最外层外部类的`NestMember`，最外层的外部类是所有内部类（包括嵌套）的`NestHost`。

声明`NestMember`使用`ClassWriter`的`visitNestMember`方法，参数是内部类的全限定名。写入它的字节码如下（仍然使用上方的代码）：

```Java
cw.visitNestMember("Test$Inner1");
cw.visitNestMember("Test$1Inner2");
cw.visitNestMember("Test$1");
```

说回到内部类文件，它也需要声明外部类和`NestHost`。声明外部类也使用`visitInnerClass`方法，需要写出所有的外部类，包括嵌套；声明`NestHost`使用`visitNestHost`方法，参数是最外层外部类全限定名。

下面是`Inner1`声明外部类的字节码写入：

```Java
cw.visitNestHost("Test");
cw.visitInnerClass("Test$Inner1", "Test", "Inner1", ACC_PRIVATE);
```

内部类的声明到此为止，接下来看看内部类和外部类的不同之处。

对于非静态内部类，它的类对象需要依托于一个外部类实例才能创建。例如下方的代码：

```Java
public class OuterClass {
  public class InnerClass {
  }
}

// ----
OuterClass outer = new OuterClass();
InnerClass inner = outer.new InnerClass(); // 正确：使用外部类实例调用内部类默认构造函数
InnerClass inner2 = new InnerClass(); // 错误：需要一个外部类对象创建内部类对象
```

非静态内部类保存了外部类的实例，保存的字段名称是`this$嵌套类深度-1`（如果名称已存在那么就在这个名字后加`$`直到不存在有这个名称的字段），以`InnerClass`举例，它的字节码实际上类似于这样：

```Java
public class InnerClass {
  
  final /* synthetic */ OuterClass this$0; 
  ...
}
```

外部类实例字段要求访问标志是`ACC_FINAL`和`ACC_SYNTHETIC`。使用`类名.this`相当于使用这个字段逐级获取，下面两个代码等价：

```Java
OuterClass.this
this.this$0
```

为了适应外部类实例字段的加入，非静态内部类的构造函数和普通的构造函数不同。它的第一个局部变量仍然是`this`，但是第二个局部变量（或者说是第一个形式参数）成为了**外部类的实例**，从第二个形式参数开始才是真正在源码层级的参数列表。内部类的默认构造函数如下（使用`InnerClass`举例）。

```Java
public InnerClass(OuterClass this$0) { // 源码中这样写和字节码中是不一样的，编译器会自动在第一个参数前加上这个参数
  // 字节码中不要求super调用是第一句语句
  this.this$0 = this$0;
  super();
}
```

字节码如下：

```Java
// cw是ClassVisitor
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "(LOuterClass;)V", null, null);
mv.visitVarInsn(ALOAD, 0);
mv.visitVarInsn(ALOAD, 1);
mv.visitFieldInsn(PUTFIELD, "OuterClass$InnerClass", "this$0", "LOuterClass;"); // 赋值字段
mv.visitVarInsn(ALOAD, 0);
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false); // 调用super
mv.visitInsn(RETURN);
mv.visitMaxs(2, 3);
mv.visitEnd();
```

## 密封类

密封类（Sealed Class）于Java 15（JEP 360）被加入，它限制了类可被哪些类继承。下面是例子：

```Java
public sealed class SupClass permits SubA, SubB { // 密封类要求至少有一个子类
}

public non-sealed class SubA extends SupClass { // 密封类子类要求必须具有non-sealed/sealed/final修饰
}

public final class SubB extends SupClass {
}
```

声明密封类属性只需要`ClassWriter`的`visitPermittedSubclass`，参数是子类全限定名。上面的`SupClass`字节码可以这样写入：

```Java
// cw是ClassWriter
cw.visitPermittedSubclass("SubA");
cw.visitPermittedSubclass("SubB");
```

---

类的基本结构就到这里，下一期：类的结构（二），主要讲解枚举、记录等特殊的类结构。