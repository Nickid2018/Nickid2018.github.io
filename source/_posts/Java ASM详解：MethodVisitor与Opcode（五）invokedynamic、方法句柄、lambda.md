---
title: Java ASM详解：MethodVisitor与Opcode（五）invokedynamic、方法句柄、lambda
layout: default
date: 2021/12/2
category:
    - Java
tags:
 - Java
 - ASM
comments: true
toc: true
---

前四篇专栏已经简要的描述了常用的字节码，这篇专栏将讲述Java 7以来最重要的字节码之一：invokedynamic。

## 方法句柄（Method Handle）

方法句柄在Java 7时被引入，位于`java.lang.invoke`包下。它类似于反射，但与反射不同的是，它的检查在创建阶段就已经结束，而反射需要每次运行时检查，所以在理论上方法引用更快。

### 方法类型（Method Type）

方法句柄包含了一个方法的信息——所在的类、名称、参数列表与返回值。为了描述参数列表与返回值，Java引入了一个类————即`MethodType`，来描述它们。

它类似于反射使用的`getMethod`方法，但是它不仅需要参数列表，它还需要返回值。创建一个MethodType可以使用下面的方法：

```Java
public static MethodType methodType(Class<?> rtype, Class<?>[] ptypes)
public static MethodType methodType(Class<?> rtype, List<Class<?>> ptypes)
public static MethodType methodType(Class<?> rtype, Class<?> ptype0, Class<?>... ptypes)
public static MethodType methodType(Class<?> rtype)
public static MethodType methodType(Class<?> rtype, Class<?> ptype0)
public static MethodType methodType(Class<?> rtype, MethodType ptypes)
```

这些方法中的`rtype`参数都代表了返回值类型，`ptypes`代表了参数类型。

下面是一个例子：使用MethodType描述`Arrays::binarySearch(Object[], int, int, Object) -> int`

```Java
MethodType mt = MethodType.methodType(int.class, Object[].class, int.class, int.class, Object.class);
```

### 从已有的方法中提取方法句柄

为了获取一个方法句柄，最简单的途径就是从一个现成的方法中提取。为了从一个现成的类中提取一个方法句柄，我们需要一个`MethodHandles.Lookup`对象，这个对象有两种获取方法：

```Java
public static Lookup lookup() // 可查找所有方法
public static Lookup publicLookup() // 只可以查找公有方法
```

`Lookup`类提供了以下方法用于查找方法句柄对象：

```Java
// 查找静态方法
public MethodHandle findStatic(Class<?> refc, String name, MethodType type) throws NoSuchMethodException, IllegalAccessException
// 查找实例方法
public MethodHandle findVirtual(Class<?> refc, String name, MethodType type) throws NoSuchMethodException, IllegalAccessException
// 查找构造函数
public MethodHandle findConstructor(Class<?> refc, MethodType type) throws NoSuchMethodException, IllegalAccessException
// 查找特殊执行的方法（需要特殊指定执行类）
public MethodHandle findSpecial(Class<?> refc, String name, MethodType type, Class<?> specialCaller) throws NoSuchMethodException, IllegalAccessException
// 查找获取字段的方法句柄（不是调用你定义的方法，而是系统内建一个getter的方法句柄）
public MethodHandle findGetter(Class<?> refc, String name, Class<?> type) throws NoSuchFieldException, IllegalAccessException
// 查找设置字段的方法句柄（同上，系统内建setter）
public MethodHandle findSetter(Class<?> refc, String name, Class<?> type) throws NoSuchFieldException, IllegalAccessException
// 查找获取静态字段的方法句柄（同上）
public MethodHandle findStaticGetter(Class<?> refc, String name, Class<?> type) throws NoSuchFieldException, IllegalAccessException
// 查找设置静态字段的方法句柄（同上）
public MethodHandle findStaticSetter(Class<?> refc, String name, Class<?> type) throws NoSuchFieldException, IllegalAccessException
```

可以看到，这些find方法都实现了某个字节码的功能：`findStatic`与`invokestatic`进行对应、`findGetter`与`getfield`对应等。

除了查找方法，从一个反射对象反反射也能获得方法句柄对象：

```Java
// 从Method反反射到一个方法句柄上
public MethodHandle unreflect(Method m) throws IllegalAccessException
// 从Method反反射到一个方法句柄上，需要指定特殊执行类
public MethodHandle unreflectSpecial(Method m, Class<?> specialCaller) throws IllegalAccessException
// 从Constructor反反射到方法句柄上
public MethodHandle unreflectConstructor(Constructor<?> c) throws IllegalAccessException
// 从Field反反射到方法句柄上
public MethodHandle unreflectGetter(Field f) throws IllegalAccessException
// 从Field反反射到方法句柄上
public MethodHandle unreflectSetter(Field f) throws IllegalAccessException
```

下面是使用例：

* 获取`System::currentTimeMillis() -> long`的方法句柄

```Java
MethodType mt = MethodType.methodType(long.class); // ()J
MethodHandle handle = MethodHandles.lookup().findStatic(System.class, "currentTimeMillis", mt);
```

* 获取获得`System.out(java.io.PrintStream)`字段的方法句柄

```Java
MethodHandle handle = MethodHandles.lookup().findStaticGetter(System.class, "out", PrintStream.class);
```

* 获取`String::<init>()`的方法句柄

```Java
MethodType mt = MethodType.methodType(void.class); // ()V
MethodHandle handle = MethodHandles.lookup().findConstructor(String.class, mt);
```

* 获取访问`sun.misc.Unsafe.theUnsafe(Unsafe)`字段的方法句柄

```Java
// 因为Unsafe.theUnsafe是private，所以要用反射先获取Field解除访问限制之后再进行反反射
Field field = Unsafe.class.getDeclaredField("theUnsafe");
field.setAccessible(true);
MethodHandle handle = MethodHandles.lookup().unreflectGetter(field);
```

### 自定义方法句柄

方法句柄不止可以通过查找获取，还可以通过`MethodHandles`内置的一些方法获取，下面是一部分内置的方法句柄生成器：

```Java
// 创建数组的方法句柄，需要传入int作为数组长度，返回数组；参数类必须是数组类；对应ANEWARRAY
public static MethodHandle arrayConstructor(Class<?> arrayClass) throws IllegalArgumentException
// 获取数组长度的方法句柄，需要一个数组传入，返回长度；参数类必须是数组类；对应ARRAYLENGTH
public static MethodHandle arrayLength(Class<?> arrayClass) throws IllegalArgumentException
// 其他数组操作忽略
// 创建返回参数自身的方法句柄
public static MethodHandle identity(Class<?> type)
// 创建返回常量的方法句柄，实际实现是identity(type).bindTo(value)
public static MethodHandle constant(Class<?> type, Object value)
// 创建返回null的方法句柄
public static MethodHandle zero(Class<?> type)
// 类似zero，但是可以传入参数
public static MethodHandle empty(MethodType type)
// 创建一个方法句柄，用于在第pos参数后插入values个参数传入target
public static MethodHandle insertArguments(MethodHandle target, int pos, Object... values)
// 创建一个方法句柄，用于取消第pos参数后valueTypes的参数传入target
public static MethodHandle dropArguments(MethodHandle target, int pos, List<Class<?>> valueTypes)
public static MethodHandle dropArguments(MethodHandle target, int pos, Class<?>... valueTypes)
// 创建取消target返回的方法句柄
public static MethodHandle dropReturn(MethodHandle target)
// 创建一个方法句柄，用于筛选第pos参数后传入target的方法参数
public static MethodHandle filterArguments(MethodHandle target, int pos, MethodHandle... filters)
```

这些生成器不止包括了基本的创建对象与对象操作，还实现了一部分流程结构，也就是说你可以通过MethodHandles“动态”地创建一个方法片段。

### 使用方法句柄

说了这么多创建方法句柄的方式，我们该怎么使用它呢？MethodHandle提供了两个方法用于执行方法句柄：

```Java
@IntrinsicCandidate
public final native @PolymorphicSignature Object invoke(Object... args) throws Throwable
@IntrinsicCandidate
public final native @PolymorphicSignature Object invokeExact(Object... args) throws Throwable
```

这两种调用方式的区别在于参数的类型转换：`invokeExact`要求参数必须准确对应MethodType定义的参数，而`invoke`会进行自动转换来尝试对应。

如果无法对应参数，这两个方法都会抛出`WrongMethodTypeException`。

下面给出一个例子，使用上面2.1创建的MethodHandle：

```Java
long time = (long) (handle.invokeExact());
```

有些情况下，我们不需要第一个参数变化（实例方法的调用对象/静态方法的第一个参数），这时我们可以用`bindTo`绑定第一个参数：

```Java
public MethodHandle bindTo(Object x)
```

下面是使用例：

```Java
MethodHandle handle = MethodHandles.lookup().findVirtual(PrintStream.class, "println", MethodType.methodType(void.class, String.class)).bindTo(System.out);
handle.invokeExact("hello");
```

说完了方法句柄，接下来来看看`CallSite`。

## 动态调用点（CallSite）

CallSite是一个为了引导`invokedynamic`字节码指向调用方法的类，通过它的`dynamicInvoker`方法可以获取一个方法句柄，这个句柄就代表了inDy的**目标**。

非常量动态调用点允许重新指定调用目标，这时inDy会对目标进行重新连接。

它有三个子类：ConstantCallSite、MutableCallSite和VolatileCallSite。它们的区别如下：

* **ConstantCallSite**指向的方法句柄不能修改，也就是永久性的。连接到它的inDy指令会永远绑定这个方法句柄。

* **MutableCallSite**允许修改指向的方法句柄目标，指向目标的行为类似普通字段。它的目标改变是不同步的——当调用目标被另一个线程修改，现在的线程不一定能同步到更新的值。为了强制同步，可以使用MutableCallSite::syncAll。连接到它的inDy指令每次调用都会调用它当前的方法句柄目标。

* **VolatileCallSite**类似MutableCallSite，其指向的目标可以修改。它的行为类似volatile字段，另一个线程修改指向目标会立刻反应到现在的线程，因此不需要syncAll之类的方法保持同步。volatile会造成不可避免的性能损失，所以如果不涉及线程问题最好用MutableCallSite。

下面演示了常量动态调用点的使用方法（此处不涉及inDy）：

```Java
MethodType type = MethodType.methodType(String.class, int.class, int.class);
MethodHandle handle = MethodHandles.lookup().findVirtual(String.class, "substring", type);
ConstantCallSite callSite = new ConstantCallSite(handle); // 创建常量调用点
MethodHandle invoker = callSite.dynamicInvoker(); // 获取动态的方法句柄
assert handle == invoker; // 这两个是一个对象
String str = (String) (invoker.invokeExact("hello", 1, 3));
```

## 引导方法（BootStrap Method，简称BSM）

在Java类执行中，少不了*动态*的东西。这些动态的东西分为两类：一种是动态计算调用点，一种是动态计算常量。引导方法就是为了它们产生的。

1. 动态计算常量，由**ConstantDynamic**表示。它们在JVM使用它们之前被解析，解析时调用的就是它内部的引导方法和它们内置的引导方法参数。

2. 动态计算调用点，也就是inDy的实现。inDy的目标在第一次调用它之前解析调用获得CallSite。

引导方法的声明有一定规则，和它们的使用方式有关：

* 如果引导方法用于动态计算常量，则引导方法的前三个参数分别是`MethodHandles.Lookup、String、Class`对象，分别代表了调用方、名称和常量类型，后面的参数是其他静态参数，返回值需要与Class对象代表的类型保持一致（或者写为Object，只需要运行时返回值可以被强制转换到指定类型就可以）。

* 如果引导方法用于动态计算调用点，则引导方法的前三个参数分别是`MethodHandles.Lookup、String、MethodType`对象，分别代表调用方、名称和调用点方法类型，后面的参数是其他静态参数，它的返回值要求是`CallSite`（通常是ConstantCallSite，当然写成Object也可以，只要保证能被强制类型转换成CallSite就不报错）

下面是一些正确的用于动态计算调用点的引导方法声明：

```Java
CallSite bootstrap(Lookup caller, String name, MethodType type, Object... args)
CallSite bootstrap(Lookup caller, String name, MethodType type) // 没有其他静态参数
CallSite bootstrap(Lookup caller, Object... nameAndType)
CallSite bootstrap(Lookup caller, String name, MethodType type, Object arg) // 只有一个静态参数
CallSite bootstrap(Lookup caller, String name, MethodType type, Object... args)
CallSite bootstrap(Lookup caller, String name, MethodType type, String... args) // 只允许String静态参数
CallSite bootstrap(Lookup caller, String name, MethodType type, String x, int y) // 只允许一个String和一个int作为静态参数传入
CallSite bootstrap(Object... args)
CallSite bootstrap(Object caller, Object... nameAndTypeWithArgs)
```

>注意：静态参数允许了动态计算常量传入。

## invokedynamic字节码

经过前面一系列的铺垫，终于我们要讲inDy该怎么写入了。

写入inDy字节码需要使用MethodVisitor的方法，`visitInvokeDynamicInsn`：

```Java
public void visitInvokeDynamicInsn(
      final String name,
      final String descriptor,
      final Handle bootstrapMethodHandle,
      final Object... bootstrapMethodArguments)
```

它的四个参数分别是名称、方法描述符、引导函数的句柄和传入引导方法的静态参数。名称和描述符都分别对应了引导方法的参数：**name**（第二个参数）、**type**（第三个参数）。

这里面的`Handle`句柄不等于MethodHandle方法句柄，但是它们也是紧密相关的，它的定义如下：

```Java
public Handle(
      final int tag,
      final String owner,
      final String name,
      final String descriptor,
      final boolean isInterface)
```

可以看到这里的参数和`visitMethodInsn`的参数基本一样。第一个参数是调用标签，分为9个，它们与方法句柄差不多：

* H_GETFIELD，对应findGetter，字节码getfield，要求isInterface是false

* H_GETSTATIC，对应findStaticGetter，字节码getstatic，要求isInterface是false

* H_PUTFIELD，对应findSetter，字节码putfield，要求isInterface是false

* H_PUTSTATIC，对应findStaticSetter，字节码putstatic，要求isInterface是false

* H_INVOKEVIRTUAL，对应findVirtual，字节码invokevirtual

* H_INVOKESTATIC，对应findStatic，字节码invokestatic

* H_INVOKESPECIAL，对应findSpecial，字节码invokespecial

* H_NEWINVOKESPECIAL，对应findConstuctor，字节码invokespecial

* H_INVOKEINTERFACE，对应findVirtual，字节码invokeinterface，isInterface是true

下面是个例子，将`Arrays::binarySearch(Object[], int, int, Object) -> int`用Handle表述：

```Java
new Handle(H_INVOKESTATIC, "java/util/Arrays", "binarySearch", "([Ljava/lang/Object;IILjava/lang/Object;)I", false);
```

那么inDy对操作栈做了什么？这就和它的第二个参数，`descriptor`有关系了。

之前说过，BSM会传入一个MethodType，而这个MethodType是用于描述返回动态调用点目标句柄的。又由于descriptor在字节码中最终会解释成为MethodType，所以能得出一个结论：descriptor决定了BSM返回CallSite内部方法句柄的类型。

而inDy在JVM的操作正是通过CallSite获取dynamicInvoker进行调用——也就是说，inDy相当于间接调用了一个类型为descriptor的方法。这样我们就不难理解inDy对操作栈干了什么：**弹出descriptor指定的一部分参数并压回规定的返回值。**

>JVM调用BSM的逻辑可以在`java.lang.invoke.BootstrapMethodInvoker`找到。

使用inDy字节码还需要一步操作：你需要让你的类访问`MethodHandles.Lookup`，因此你需要在类声明时加入一个`visitInnerClassInsn`（其实不加也不会报错，但是最好加上）：

```Java
cw.visitInnerClass("java/lang/invoke/MethodHandles$Lookup", "java/lang/invoke/MethodHandles", "Lookup", ACC_PUBLIC | ACC_FINAL | ACC_STATIC);
```

## lambda表达式

匿名函数表达式，简称lambda表达式，它在Java 8被加入。它简化了一部分的匿名类，让代码更加简洁。

为了展示它的用法和字节码表示，我们先定义一个接口和一个方法：

```Java
public static void test(StringSupplier lambda) {
	System.out.println(lambda.getString());
}

@FunctionalInterface
public interface StringSupplier {
	
	String getString();
}
```

接着，我们使用这个方法：

```Java
public static void main(String[] args) {
	test(() -> "hello");
}
```

这时，后面的`() -> "hello"`被解析成了一个`StringSupplier`的实现类对象。但是，在字节码中无法自动去生成一个这样的类用于适配它。于是，javac在此处写入了inDy字节码要求动态生成。

动态生成lambda调用点的引导方法位于`java.lang.invoke.LambdaMetafactory`：

```Java
// 使用了优化协议的标准版本
public static CallSite metafactory(MethodHandles.Lookup caller,
                                   String interfaceMethodName,
                                   MethodType factoryType,
                                   MethodType interfaceMethodType,
                                   MethodHandle implementation,
                                   MethodType dynamicMethodType)
            throws LambdaConversionException
// 备用版本
public static CallSite altMetafactory(MethodHandles.Lookup caller,
                                   String interfaceMethodName,
                                   MethodType factoryType,
                                   Object... args)
            throws LambdaConversionException
```

通常情况下，javac生成的lambda都是通过第一个BSM的，这6个参数的意义分别是：

* **caller**，由JVM提供的查找对象，lambda会使用这个进行动态类创建

* **interfaceMethodName**，lambda实现接口内部需要实现的方法名称

* **factoryType**，要求BSM返回CallSite内部指向方法句柄的方法类型

* **interfaceMethodType**，lambda实现接口内需要实现方法的类型

* **implementation**，实现lambda内部代码功能的方法句柄

* **dynamicMethodType**，实现lambda内部代码功能方法的类型，和interfaceMethodType相同或者是它的更具体的类型

可以看到，为了提供lambda的功能，javac会让inDy字节码连接到另一个方法上去。这种方法不需要我们自己写，它是编译时自动生成的，名称是`lambda$方法名$序号`。上例中，javac动态生成的lambda方法如下：

```Java
private static /* synthetic */ String lambda$main$0() {
	return "hello";
}
```

这些方法都带有`private`和`synthetic`的访问标志，是否拥有`static`访问标志取决于lambda在的方法是否静态和是否使用this对象。

接下来，我们使用这个方法连接到LambdaMetafactory：

```Java
// 注意：这只是个演示！真正的字节码不是这样，这只是一种方式用于理解调用流程！
test(LambdaMetafactory.metafactory(
	lookup, // JVM提供
	"getString", // lambda实现方法的名称
	MethodType.methodType(StringSuppiler.class), // 返回CallSite中的方法类型
	MethodType.methodType(String.class), // 需要提供接口方法的类型，描述符()Ljava/lang/String;
	MethodHandles.lookup().findStatic(Test.class, "lambda$main$0", MethodType.methodType(String.class)), // lambda的实现
	MethodType.methodType(String.class)) // 指向方法目标的类型，描述符()Ljava/lang/String;
	.dynamicInvoker()
	.invokeExact()
);
```

`metafactory`通过这些参数可以动态创建一个类实现指定的接口获得实现接口的对象。具体而言，它通过asm库（java内置了asm库）在现在的类中动态的生成了内部类，类的名称是`$Lambda$序号`（但是在`getClass()`获取时名称不是这个，因为这个类被“隐藏”定义，会带上另一个编号）。对于这个例子，生成的内部类像下面这样：

```Java
final /* synthetic */ class $Lambda$14 implements StringSupplier { // 序号不重要
	
	private $Lambda$14() {
		super();
	}

	public String getString() { // 实现
		return Test.lambda$main$0(); // 注意这里使用的不是方法句柄
	}
}
```

对于这种lambda表达式，生成的类对象永远不变，所以JVM对此进行优化——这种lambda只会生成一个实例，返回的CallSite其实只是返回一个常量（详情可见`InnerClassLambdaMetafactory`）。

说回到字节码的写入。之前说过visitInvokeDynamicInsn和BSM的参数一一对应，所以我们可以这样写入：

```Java
mv.visitInvokeDynamicInsn("getString", // 实现的方法名称
                          "()LStringSupplier;", // 要求返回CallSite的类型
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false), // BSM的句柄
                          Type.getType("()Ljava/lang/String;"), // 接口方法的类型
                          new Handle(H_INVOKESTATIC, "Test", "lambda$main$0", "()Ljava/lang/String;", false), // lambda的实现代码
                          Type.getType("()Ljava/lang/String;") // lambda实现方法的类型
);
mv.visitMethodInsn(INVOKESTATIC, "Test", "test", "(LStringSupplier;)V", false); // 调用test
mv.visitInsn(RETURN);
mv.visitMaxs(1, 1); // 最大操作栈[StringSupplier]，局部变量表[[Ljava/lang/String;]
```

除了这种lambda外，还有另一种lambda：它们需要局部变量传入内部。这些局部变量有要求——它们无法被修改，或者叫“等效终态”。下面是一个例子：

```Java
// main方法
/* final */ String hello = "hello";
test(() -> hello);
```

由于传入了局部变量，lambda的实现方法就需要多加一个参数用于传递这个变量。下面是javac生成的lambda代理实现方法：

```Java
private static /* synthetic */ String lambda$main$0(String str) {
	return str;
}
```

但是这个str要怎么透过inDy字节码进行传入？JVM为了解决这个问题，在动态生成的委托类上做了一些操作：让传入的变量先用构造函数存储在字段里，在调用时取出字段值：

```Java
final /* synthetic */ class $Lambda$14 implements StringSupplier { // 序号不重要
	
	private final String arg$0; // 用于储存传入的局部变量，名称以序号命名

	private $Lambda$14(String str) {
		super();
		arg$0 = str;
	}

	public String getString() { // 实现
		return Test.lambda$main$0(arg$0);
	}
}
```

但是，这种lambda的CallSite不能返回一个常量——因为我们不能保证局部变量是同一个值！因此，这个CallSite内部指向了动态生成内部类的构造函数。

接下来，我们用字节码写入一下：

```Java
mv.visitLdcInsn("hello");
mv.visitVarInsn(ASTORE, 1); // 保存变量到1号位
mv.visitVarInsn(ALOAD, 1); // 加载1号位
mv.visitInvokeDynamicInsn("getString", // 实现的方法名称
                          "(Ljava/lang/String;)LStringSupplier;", // 要求返回CallSite的类型，注意这里要求传入String局部变量
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false), // BSM的句柄
                          Type.getType("()Ljava/lang/String;"), // 接口方法的类型
                          new Handle(H_INVOKESTATIC, "Test", "lambda$main$0", "(Ljava/lang/String;)Ljava/lang/String;", false), // lambda的实现代码
                          Type.getType("()Ljava/lang/String;") // lambda实现方法的类型（生成内部类实现方法的类型）
);
mv.visitMethodInsn(INVOKESTATIC, "Test", "test", "(LStringSupplier;)V", false); // 调用test
mv.visitInsn(RETURN);
mv.visitMaxs(1, 2); // 最大操作栈[StringSupplier]，局部变量表[[Ljava/lang/String;Ljava/lang/String;]
```

## 方法引用

当lambda内只有一行方法调用时，在特定条件下可以简写为方法引用。它分为不同的类型：

### 静态调用

当方法引用指向一个类中的静态方法时，就是静态调用，类似于：

```Java
public static String hello() {
	return "hello";
}

// ...
test(Test::hello);
```

它的实现类似于lambda，但是不同的是，javac编译时不会生成一个新的方法用于lambda定位，而是选择直接指向这个方法：

```Java
mv.visitInvokeDynamicInsn("getString", // 实现的方法名称
                          "()LStringSupplier;", // 要求返回CallSite的类型
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false), // BSM的句柄
                          Type.getType("()Ljava/lang/String;"), // 接口方法的类型
                          new Handle(H_INVOKESTATIC, "Test", "hello", "()Ljava/lang/String;", false), // 方法引用的目标
                          Type.getType("()Ljava/lang/String;") // lambda实现方法的类型（生成内部类实现方法的类型）
);
mv.visitMethodInsn(INVOKESTATIC, "Test", "test", "(LStringSupplier;)V", false); // 调用test
mv.visitInsn(RETURN);
mv.visitMaxs(1, 1); // 最大操作栈[StringSupplier]，局部变量表[[Ljava/lang/String;]
```

### 对象调用

当方法引用的目标不是静态的，它就需要使用一个对象用于方法的调用，下面是个例子：

```Java
public String hello() {
	return "hello";
}

// ...
test(new Test()::hello);
```

这类似于将局部变量传入了lambda内部，因此这里的inDy字节码是这样写的：

```Java
mv.visitTypeInsn(NEW, "Test"); // 创建Test类对象
mv.visitInsn(DUP); // 复制一份
mv.visitMethodInsn(INVOKESPECIAL, "Test", "<init>", "()V", false); // 构造函数调用
mv.visitInvokeDynamicInsn("getString", // 实现的方法名称
                          "(LTest;)LStringSupplier;", // 要求返回CallSite的类型，注意这里要求传入Test局部变量
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false), // BSM的句柄
                          Type.getType("()Ljava/lang/String;"), // 接口方法的类型
                          new Handle(H_INVOKEVIRTUAL, "Test", "hello", "()Ljava/lang/String;", false), // 方法引用的目标
                          Type.getType("()Ljava/lang/String;") // lambda实现方法的类型（生成内部类实现方法的类型）
);
mv.visitMethodInsn(INVOKESTATIC, "Test", "test", "(LStringSupplier;)V", false); // 调用test
mv.visitInsn(RETURN);
mv.visitMaxs(2, 1); // 最大操作栈[Test,Test]，局部变量表[[Ljava/lang/String;]
```

对象调用的对象没有特殊要求，只需要能获得这个局部变量就可以。方法引用的目标可以是实例方法，也可以是抽象方法（区别在于Handle的标签）。

由于JVM不能保证传入的局部变量是非空的（例外就是上面的情况：直接新建对象），所以在传入lambda之前，JVM会进行`requireNonNull`进行检查。也就是说，下面这两种方式等价：

```Java
Test obj;
// ...
// 1
test(obj::hello);
// 2
Objects.requireNonNull(obj);
test(() -> obj.hello());
```

上面的方法引用版本的代码可以写为：

```Java
mv.visitVarInsn(ALOAD, 1); // 假设位于1号位
mv.visitInsn(DUP); // 复制一份，使用时可以不进行复制
mv.visitMethodInsn(INVOKESTATIC, "java/util/Objects", "requireNonNull", "(Ljava/lang/Object;)Ljava/lang/Object;", false);
mv.visitInsn(POP); // 弹出栈顶
mv.visitInvokeDynamicInsn("getString", // 实现的方法名称
                          "(LTest;)LStringSupplier;", // 要求返回CallSite的类型，注意这里要求传入Test局部变量
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false), // BSM的句柄
                          Type.getType("()Ljava/lang/String;"), // 接口方法的类型
                          new Handle(H_INVOKEVIRTUAL, "Test", "hello", "()Ljava/lang/String;", false), // 方法引用的目标
                          Type.getType("()Ljava/lang/String;") // lambda实现方法的类型（生成内部类实现方法的类型）
);
mv.visitMethodInsn(INVOKESTATIC, "Test", "test", "(LStringSupplier;)V", false); // 调用test
mv.visitInsn(RETURN);
mv.visitMaxs(2, 1); // 最大操作栈[Test,Test]，局部变量表[[Ljava/lang/String;]
```

除了这两种方式外，我们还能使用超类的实例方法，现在假设Test继承于`SuperTest`，有一个superTest方法，**这时javac不会直接引用超类方法，而是生成lambda实现方法在内部调用`invokespecial`**。下面是使用例：

```Java
test(super::superTest);
```

javac生成的lambda实现方法是这样的：

```Java
private /* synthetic */ String lambda$main$0() {
	return super.superTest();
}
```

接下来的代码省略（因为和上面一样）。

最后，还有一种对象调用：lambda内部传入了一个对象，我们可以通过这个对象进行调用。这个调用方式和静态调用差不多，只不过Handle的标签是`H_INVOKEVIRTUAL`，这个也不举例了。

### 构造函数调用

方法引用允许传递构造函数，下面使用了String的无参构造函数传入test内部：

```Java
test(String::new);
```

这时，传入的方法句柄是构造函数，对应了Handle中的`H_NEWINVOKESPECIAL`：

```Java
mv.visitInvokeDynamicInsn("getString", // 实现的方法名称
                          "()LStringSupplier;", // 要求返回CallSite的类型
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false), // BSM的句柄
                          Type.getType("()Ljava/lang/String;"), // 接口方法的类型
                          new Handle(H_NEWINVOKESPECIAL, "String", "<init>", "()V", false), // 方法引用的目标
                          Type.getType("()Ljava/lang/String;") // lambda实现方法的类型（生成内部类实现方法的类型）
);
mv.visitMethodInsn(INVOKESTATIC, "Test", "test", "(LStringSupplier;)V", false); // 调用test
mv.visitInsn(RETURN);
mv.visitMaxs(1, 1); // 最大操作栈[LStringSupplier;]，局部变量表[[Ljava/lang/String;]
```

### 数组构造调用

除了普通的构造函数，数组也可以通过方法引用创建。它只需要一个int作为参数，因此它实现的方法必须只有一个int形参：

```Java
public static void test(ArrayIntSupplier lambda) {
	// ...
}

@FunctionalInterface
public interface ArrayIntSupplier {
	
	Object[] getString(int count);
}

// ...
test(Object[]::new);
```

这种方法引用也不是直接指向构造函数的，还是javac生成lambda实现方法并引用的：

```Java
private static /* synthetic */ Object[] lambda$main$0(int cnt) {
	return new Object[cnt];
}
```

到此，所有方法引用的写入方式就都介绍完了。

## 字符串连接

在学习Java的时候，我们就知道Java的String允许用`+`进行连接。但是，Java没有符号重载，那么字符串是怎么打破这个限制的呢？答案就是javac编译时做了一些“操作”。

接下来，我们使用这个例子：

```Java
"hello world at " + System.currentTimeMillis()
```

在Java 8，字符串的连接被自动识别为`StringBuilder`的链式调用，那么上面的这句话在javac编译之后就变成了这样：

```Java
new StringBuilder().append("hello world at ").append(System.currentTimeMillis()).toString()
```

字节码写入如下：

```Java
mv.visitTypeInsn(NEW, "java/lang/StringBuilder");
mv.visitInsn(DUP);
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false); // 构造函数
mv.visitLdcInsn("hello world at "); // 字符串常量
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false); // append
mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
```

但是这种方式有两个缺点：一是会生成大量的字节码片段，使类文件膨胀；二是这种调用每次都会生成StringBuilder对象，性能会损失一部分。

所以，从Java 9开始，字符串连接使用inDy字节码动态调用。它使用的引导方法位于`StringConcatFactory`。

```Java
public static CallSite makeConcat(MethodHandles.Lookup lookup,
                                  String name,
                                  MethodType concatType) throws StringConcatException
public static CallSite makeConcatWithConstants(MethodHandles.Lookup lookup,
                                               String name,
                                               MethodType concatType,
                                               String recipe,
                                               Object... constants)
        throws StringConcatException
```

`makeConcat`是`makeConcatWithConstants`的简化版本，如果没有常量，就用第一个方法，但是javac编译时通常使用第二个方法，所以我们对它进行讲解。

首先说说方法参数的意义：

* **lookup**，由JVM提供的查找对象

* **name**，名称，和最后的连接效果没有任何关系，只要不是null都能传入。程序写入常用`makeConcatWithConstants`

* **concatType**，生成CallSite的签名，返回值需要是String，参数列表要和字符串中的变量的数量、类型和位置保持一致

* **recipe**，用于连接字符串的模板，只有两种字符：`\u0001`代表了这里应该写入变量，`\u0002`代表这里应该写入常量。\u0001的数量、位置需要和变量保持一致；\u0002的数量、位置要与常量保持一致

* **constants**，字符串中的常量部分，数量和\u0002一致，可以不是String。

它的原理比lambda要简单——它是动态生成了一个MethodHandle存储到CallSite中，因此在执行一次BSM之后它就成为了常量。

现在我们用它写入字节码：

```Java
mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false); // 先获得字符串中的变量
mv.visitInvokeDynamicInsn("makeConcatWithConstants", // name随意，只要求不是null
                          "(J)Ljava/lang/String;", // 变量只有一个long，要求返回String
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/StringConcatFactory", "makeConcatWithConstants", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;", false), // BSM句柄，注意不定长参数需要写成数组
                          "hello world at \u0001" // 模板
);
```

但是你可能有一个疑问：如果我字符串里面本身有\u0001或者\u0002不就出错了吗？JVM考虑了这个情况，它的解决方案是——提取这一段字符串为常量放到后面。例如下面这个字符串：

```Java
"\u0001 hello" + System.currentTimeMillis()
```

它的写入是：

```Java
mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false); // 先获得字符串中的变量
mv.visitInvokeDynamicInsn("makeConcatWithConstants", // name随意，只要求不是null
                          "(J)Ljava/lang/String;", // 变量只有一个long，要求返回String
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/StringConcatFactory", "makeConcatWithConstants", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;", false), // BSM句柄，注意不定长参数需要写成数组
                          "\u0002\u0001", // 模板，常量+变量
                          "\u0001 hello" // 常量的值
);
```

## 模式匹配

在Java 17，模式匹配进行了预览。下面就是它的使用例：

```Java
Object o ...
switch (o) {
	case String s:
		System.out.println(s.substring(1));
		break;
	case Number n:
		System.out.println(n.longValue());
		break;
	default:
		System.out.println("Others");
}
```

模式匹配之间必须使用`break`，否则会被提示为非法。

使用增强型switch可以写成下面形式：

```Java
Object o ...
switch (o) {
	case String s -> System.out.println(s.substring(1));
	case Number n -> System.out.println(n.longValue());
	default -> System.out.println("Others");
}
```

它的写入和其他switch不同，它使用了inDy用于获得序号，再用这个序号进行`lookupswitch`。

它使用的引导方法位于`java.lang.runtime.SwitchBootstraps`：

```Java
public static CallSite typeSwitch(MethodHandles.Lookup lookup,
                                  String invocationName,
                                  MethodType invocationType,
                                  Object... labels)
```

它的参数意义如下：

* **lookup**，由JVM提供的查找对象

* **invocationName**，名称，和最后的效果没有任何关系，只要不是null都能传入。程序写入常用`typeSwitch`

* **invocationType**，要求第一个参数非基本类型、第二个参数是int、返回值是int的方法类型，也就是说它强制要求传入一个对象和一个int。对象用于检查模式，int用于确定lookupswitch的起始位置（通常是0）。返回值是从第二个参数开始的连续数列中的一个值。

* **labels**，模式匹配目标。可以是Class、Integer或String对象，但是实际上编译时只使用了Class对象。它规定了返回的CallSite的内容：如果输入的对象是这个Class的对象，返回对应的位置。如果输入的对象是null，返回-1。如果输入的对象不存在匹配项，返回labels数组的长度。

通过这个引导方法，上面的代码可以变为：

```Java
Object o ...
Objects.requireNonNull(o); // 先检查对象是否为null
Object tmp = o; // 复制一份，javac是这么做的
int startIndex = 0; // 传入的起始lookup偏移
// --- 这里有一次栈帧信息写入 --- labelStart
// 这里加载了tmp和startIndex
switch(#invokedynamic) { // inDy的调用部分省略
	case 0:
		/* labelString */
		String s = (String) o;
		System.out.println(s.substring(1));
		break;
	case 1:
		/* labelNumber */
		Number n = (Number) n;
		System.out.println(n.longValue());
		break;
	default:
		/* labelDefault */
		System.out.println("Others");
}
/* labelEnd */
```

使用字节码写入：

```Java
mv.visitVarInsn(ALOAD, 0); // o
mv.visitInsn(DUP); // 复制一份，同样，不复制也行，这是javac的迷惑操作
mv.visitMethodInsn(INVOKESTATIC, "java/util/Objects", "requireNonNull", "(Ljava/lang/Object;)Ljava/lang/Object;", false); // 检查null
mv.visitInsn(POP); // 弹出栈顶
mv.visitVarInsn(ASTORE, 1); // 转存到临时变量tmp
mv.visitInsn(ICONST_0); // 常量0
mv.visitVarInsn(ISTORE, 2); // 给局部变量startIndex赋值
Label labelStart = new Label(); // 用于标记switch块的开始（这样tmp和startIndex不能在结束时访问）
mv.visitLabel(labelStart);
mv.visitFrame(F_APPEND, 2, new Object[]{ "java/lang/Object", INTEGER }, 0, null); // 写入栈帧信息
mv.visitVarInsn(ALOAD, 1); // 加载对象
mv.visitVarInsn(ILOAD, 2); // 加载偏移
mv.visitInvokeDynamicInsn("typeSwitch", // 无关紧要的名字，只要不是null就行
                          "(Ljava/lang/Object;I)I", // CallSite的方法签名
                          new Handle(H_INVOKESTATIC, "java/lang/runtime/SwitchBootstraps", "typeSwitch", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;", false), // BSM句柄
                          Type.getType("Ljava/lang/String;"), // 第一个模式
                          Type.getType("Ljava/lang/Number;") // 第二个模式
);
Label labelString = new Label(), labelNumber = new Label(), labelDefault = new Label();
mv.visitLookupSwitchInsn(labelDefault, new int[]{ 0, 1 }, new Label[]{ labelString, labelNumber }); // 返回值用于lookupSwitch
mv.visitLabel(labelString);
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitVarInsn(ALOAD, 1); // tmp
mv.visitTypeInsn(CHECKCAST, "java/lang/String"); // 强制类型转换
mv.visitVarInsn(ASTORE, 3); // 存储到s
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitVarInsn(ALOAD, 3);
mv.visitInsn(ICONST_1);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/String", "substring", "(I)Ljava/lang/String;", false);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
Label labelEnd = new Label();
mv.visitJumpInsn(GOTO, labelEnd); // break
mv.visitLabel(labelNumber);
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitVarInsn(ALOAD, 1); // tmp
mv.visitTypeInsn(CHECKCAST, "java/lang/Number"); // 强制类型转换
mv.visitVarInsn(ASTORE, 4); // 存储到n
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitVarInsn(ALOAD, 4);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Number", "longValue", "()J", false);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(J)V", false);
mv.visitJumpInsn(GOTO, labelEnd); // break
mv.visitLabel(labelDefault);
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("Others");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitLabel(labelEnd);
mv.visitFrame(F_CHOP, 2, null, 0, null); // 清除局部变量
mv.visitInsn(RETURN);
```

>通过SwitchBootstrap可以看出switch以后可能将尽可能使用invokedynamic：typeSwitch支持String输入，也许之后会将String的switch语句修改为这种实现；现在的类内部还有一个enumSwitch但是javac并不能编译出这个引导方法。在下个版本也许会进一步增加细节。

## 自定义引导方法

<font color = "red" >注意：自定义一个引导方法可能导致你的程序不稳定、出现奇奇怪怪的问题、编译变得极度麻烦。如果不是特殊用途（比如说真正的让一个反编译器完全失效）不要用这个！</font>

根据上面引导方法的定义和inDy的实现，我们自己也能创造出一个引导方法——只需要满足要求就好。下面就是一个简单的引导方法：

```Java
public static CallSite bootstrap(MethodHandles.Lookup lookup, String name, MethodType type) {
	if (!type.returnType().equals(String.class) || type.parameterCount() != 0)
		throw new RuntimeException("Unknown"); // 我们限定只能输入这种方法类型
	return new ConstantCallSite(MethodHandles.constant(String.class, "hello")); // 方法句柄：返回常量hello
}
```

接着，我们在我们的方法里面用字节码指向它：

```Java
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"); // System.out
mv.visitInvokeDynamicInsn("testBootstrap", // 名字，我们定义的引导方法没有使用它
                          "()Ljava/lang/String;", // 方法类型，我们强制规定了使用返回值为String的无参方法
                          new Handle(H_INVOKESTATIC, "Test", "bootstrap", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false) // 引导方法的句柄
);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false); // println
mv.visitInsn(RETURN);
mv.visitMaxs(2, 0); // 最大操作栈是[java/io/PrintStream,java/lang/String] 没有局部变量
```

接下来就是执行，输出结果是：

```shell
hello
```

这样，我们就成功让字节码指向我们自定义的引导方法。

全部的代码：<https://paste.ubuntu.com/p/d82FNcP6jS/>

## 动态常量（Constant Dynamic）

之前在BSM那里简单提到了动态计算常量，这是JEP 309（Java 11）引入的，在这里我们再进一步深入讲解。

首先，它的BSM定义和动态调用点的BSM定义方式不同，详情可以看上面。在写入ASM时，它使用的是`visitLdcInsn`，和普通常量一样。

创建一个动态常量使用ConstantDynamic，它的构造函数如下：

```Java
public ConstantDynamic(
      final String name,
      final String descriptor,
      final Handle bootstrapMethod,
      final Object... bootstrapMethodArguments)
```

可以看到它和visitInvokeDynamicInsn差不多，唯一的区别是：`descriptor`是类描述符而不是方法描述符。因此，所有动态常量的BSM都不允许传入变量。

有关于动态常量的BSM都存储到了一个类中：`java.lang.invoke.ConstantBootstraps`。

```Java
// null的常量，只与type有关
public static Object nullConstant(MethodHandles.Lookup lookup, String name, Class<?> type)
// 获得类描述符为name的Class对象，type必须是Class.class
public static Class<?> primitiveClass(MethodHandles.Lookup lookup, String name, Class<?> type)
// 获得名称为name的枚举对象
public static <E extends Enum<E>> E enumConstant(MethodHandles.Lookup lookup, String name, Class<E> type)
// 获得位于declaringClass内部的静态终态字段name
public static Object getStaticFinal(MethodHandles.Lookup lookup, String name, Class<?> type, Class<?> declaringClass)
// 上面的简化版本，令declaringClass与type相同
public static Object getStaticFinal(MethodHandles.Lookup lookup, String name, Class<?> type)
// 获得handle使用静态参量args的结果
public static Object invoke(MethodHandles.Lookup lookup, String name, Class<?> type, MethodHandle handle, Object... args)
// 获得字段的VarHandle
public static VarHandle fieldVarHandle(MethodHandles.Lookup lookup, String name, Class<VarHandle> type, Class<?> declaringClass, Class<?> fieldType)
// 获得静态字段的VarHandle
public static VarHandle staticFieldVarHandle(MethodHandles.Lookup lookup, String name, Class<VarHandle> type, Class<?> declaringClass, Class<?> fieldType)
// 获得数组的VarHandle
public static VarHandle arrayVarHandle(MethodHandles.Lookup lookup, String name, Class<VarHandle> type, Class<?> arrayClass)
// 将value显式转换到dstType
public static Object explicitCast(MethodHandles.Lookup lookup, String name, Class<?> dstType, Object value) throws ClassCastException
```

使用动态计算常量可以使用其他动态计算常量作为静态参数，这时JVM会倒序一个个计算创建常量。

下面是个例子：

```Java
#ldc 动态常量 System.out
.println("hello");
```

字节码写入如下：

```Java
mv.visitLdcInsn(new ConstantDynamic(
                "out", // 字段的名称
                "Ljava/io/PrintStream;", // 字段类型
                new Handle(H_INVOKESTATIC, "java/lang/invoke/ConstantBootstraps", "getStaticFinal", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/Class;Ljava/lang/Class;)Ljava/lang/Object;", false), // BSM句柄
                Type.getType("Ljava/lang/System;") // 字段声明位置
));
mv.visitLdcInsn("hello");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
```

每个ConstantDynamic都可以复用——你可以使用一个对象传入到不同的LDC里面去。这些对象最终和普通常量一样存储到常量池内部。

***

这篇专栏就这些了，只讲了一个字节码，但是内容很多。加上以前的一共191个。

有错误可以在评论区指出。