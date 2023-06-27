---
title: Java ASM详解：泛型
layout: default
date: 2022/2/13
category:
	- Java
tags:
 - Java
 - ASM
comments: true
toc: true
---

在之前的字节码学习中，我们都是对有确定类型的数据/类进行操作。可是，Java有着一种“黑”操作用于更好的检查对象类型：在Java 5加入的泛型。

## 泛型的声明

在正式导入泛型声明之前，先说说泛型描述符。

普通的非基本类型的描述符的命名方法是`L+类名+;`，但是泛型不能以这种方式命名——它需要和普通的描述符有区别以便于区分。所以，它的命名要求是`T+泛型名称+;`。所以一个命名为`T`的泛型的描述符是`TT;`。接下来说回到声明。

每个泛型在使用之前必须经过声明，声明的位置是类、方法、字段、内部类等的声明部分。如果非静态方法或字段使用的泛型在它们所在的类中被声明过了，那么在它们声明时这个泛型不需要二次声明。

泛型要求必须继承于一个确切的类，对于没有写`extends`限定的泛型，它们默认继承于`Object`。

在字节码中，泛型的声明要用尖括号包围。每对泛型用冒号隔开，冒号前方是类型参数名称，后方是类型参数的超类或另一个已声明的类型参数。如果类型参数继承于一个接口，那么应该使用双冒号。（可以不用遵守这个约定，冒号数量不影响解析）

泛型与泛型之间不需要多余的分隔符分割它们的定义，这是因为描述符都以分号结尾从而阻止二义性的解析。（这里也就说明了为什么基本类型不能当类型参数的限制符号，它们的描述符不带分号会造成不可阻止的二义性解析）

下面是泛型声明的例子：

```Java
T extends InputStream
	<T:Ljava/io/InputStream;>
T extends Serializable
	<T::Ljava/io/Serializable;>
T, E extends T
	<T:Ljava/lang/Object;E:TT;>
```

但是上面的几条规范不能包括我们使用的所有情况。就比如下面的例子：

```Java
T extends Enum<T>
```

类型参数的超类具有类型参数的使用，这时我们需要写出超类类型参数的位置和限定关系。

对于一个使用了类型参数的类，它需要在类名之后分号之前用尖括号表示各个类型参数的配置。每个参数都可以是确切的类、声明过的类型参数或通配符。和上面的声明一样，因为描述符以分号结尾，不会产生二义性解析，因此不需要额外分隔符用于分割定义。

对于上面的例子，它的声明就可以表示成：

```Java
<T:Ljava/lang/Enum<TT;>;>
```

这个T可以代指所有的枚举类型，因为所有的枚举都隐式继承`Enum`。

如果超类类型参数使用通配符`?`代替，就有三种情况：

* 没有任何限定，需要用`*`填入

* 有extends限定，需要用`+`加上超类描述符填入

* 有super限定，需要用`-`加上子类描述符填入

下面是这三种情况的实例：

```Java
T extends Set<?>
	<T::Ljava/util/Set<*>;>
T extends List<? extends T>
	<T::Ljava/util/List<+TT;>;>
T extends Map<? super FileInputStream, T>
	<T::Ljava/util/Map<-Ljava/io/FileInputStream;TT;>;>
```

## 泛型签名

泛型在声明之后就可以用于描述类、字段、方法等的具体描述符，这部分也叫泛型签名，是另一种描述类、字段、方法类型的方式。

泛型签名与修饰的结构有关。它不仅包含了类、字段、方法每个具体位置上需要的具体类型，还包含对泛型的声明。如果这个签名中使用的泛型没有被声明过，那么就应该在签名的前方加入它的声明。（也就是说声明是签名的一部分）

下面对签名的不同作用位置分开说明：

* 泛型签名修饰了一个类。类有两个地方需要泛型的信息：超类和实现接口。这些类的具体泛型信息要以超类和实现接口的顺序排列写入，写入规则和上面的类型参数超类写入规则一样。

```Java
public class Test<T> extends SuperClass<T> implements Interface1, Interface2<T>
	<T:Ljava/lang/Object;>LSuperClass<TT;>;LInterface1;LInterface2<TT;>;
```

* 泛型签名修饰了一个字段。字段只需要描述字段本身具体的泛型信息，并且字段不能定义泛型，所以泛型签名只包含它的类型的泛型信息。

```Java
public Set<T> set;
	Ljava/util/Set<TT;>;
```

* 泛型签名修饰了一个方法。这时这里就是方法描述符的具体泛型信息。格式类似普通的方法描述符，只是泛型信息的写入要遵照上方规则。如果方法带有抛出异常声明并且异常列表含有泛型，那么在描述符之后还要加上异常列表。异常列表的每个类都要用^开头并且需要写出具体的泛型信息。

```Java
public <E extends Exception> void test(T obj, List<? super T> list) throws E
	<E:Ljava/lang/Exception;>(TT;Ljava/util/List<-TT;>;)V^TE;
```

下面我们定义一个类作为例子，写出所有成员的泛型签名：

```Java
public abstract class Test<T extends List<? extends Number>> implements Comparable<T> {
	// <T::Ljava/util/List<+Ljava/lang/Number;>;>Ljava/lang/Object;Ljava/lang/Comparable<TT;>

	public abstract <E extends Exception> void test1(T obj) throws E;
	// <E:Ljava/lang/Exception;>(TT;)V^TE;

	public abstract T test2();
	// ()TT;

	public Set<T> set;
	// Ljava/lang/Set<TT;>;
}
```

## 泛型擦除

在阅读完上面的文本后，你可能会有一个疑问：Java已经有方法描述符可以用来描述方法、字段描述符来描述字段等等，为什么还要再加入一个泛型签名用于额外的检查呢？这就有关于Java对泛型的具体实现方式，也就是**泛型擦除**。

在正式介绍这个机制之前，我们先看看反射对于泛型的处理。定义下面这个类：

```Java
public class Test<T extends InputStream> {
	
	public void test(T stream) {
	}
}
```

接下来我们想要反射调用`test`这个方法。可是我们看到这个方法的形参列表里面含有一个类型参数T，这个我们没有办法具体表示。所以，我们可以用`Class`类的`getDeclaredMethods`获取所有方法检查它的真正形参列表的样子：

```Java
[public void Test.test(java.io.InputStream)]
```

可以看到，类型参数的形参的位置上使用了它的超类`InputStream`。字节码在所有的含有类型参数的地方都用它们的超类的原始类型代替，这种现象就是**泛型擦除**。

泛型擦除保证了JVM获取方法时不含有未知量，在本质上其实是保证各个字节码中只存在静态的信息，这样能保证运行的正确性，不会产生方法的二义性调用等。

对于具体的字节码，所有字节码指令都不能用带有泛型信息的类。具体来说，`new/checkcast/instanceof/invokeXXX`字节码都不能使用任何泛型信息，它们只能使用原始类型和替换的超类。

在JVM中，泛型擦除之后泛型的真正实现其实是`checkcast`等字节码指令的约束和运行时对于对象真实类型推断的方法。在下面的例子中能体现字节码对于泛型的约束。

## 实例：泛型方法

下面，我们要生成这样的代码：

```Java
public static <T extends Number> int test(List<? extends T> list, T value) {
	Comparator<? super Number> comparator = (n1, n2) -> (int) Math.signum(n1.doubleValue() - n2.doubleValue());
	list.sort(comparator);
	T num = list.get(0);
	return num.intValue() + Collections.binarySearch(list, value, comparator);
}
```

首先写出它的方法描述符和泛型签名：

```Java
Desc:
	(Ljava/util/List;Ljava/lang/Number;)I
	// List擦除类型参数所以回到原始类型，T擦除到父类Number
Signature:
	<T:Ljava/lang/Number;>(Ljava/util/List<+TT;>;TT;)I
```

接下来注意到第一行有一个lambda表达式，它实现的是`Comparator`类的`compare`方法。根据`comparator`变量的类型，得知实现方法的参数列表应该是`(Number, Number)`。所以我们应该生成下面的lambda方法：

```Java
private static /* synthetic */ int lambda$test$0(Number n1, Number n2) {
	return (int) Math.signum(n1.doubleValue() - n2.doubleValue());
}
```

字节码写入如下：

```Java
// 省略方法声明，mv是MethodVisitor
mv.visitVarInsn(ALOAD, 0); // n1
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Number", "doubleValue", "()D", false);
mv.visitVarInsn(ALOAD, 1); // n2
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Number", "doubleValue", "()D", false);
mv.visitInsn(DSUB); // 相减
mv.visitMethodInsn(INVOKESTATIC, "java/lang/Math", "signum", "(D)D", false);
mv.visitInsn(D2I); // 强制类型转换
mv.visitInsn(IRETURN);
mv.visitMaxs(4, 2);
```

这时候就能把comparator给构造出来了。注意我们实现的方法是`compare(Object, Object)`——类型参数T被擦除到父类型`Object`（**这和你代码中的使用无关，取决于类和方法的定义**），而我们需要的实现是`(Number, Number)`，这两个不冲突，但是会损失类型信息（下文会讲）。因此我们inDy的参数第一个是实现的目标的描述符，第三个是我们真正实现目标的描述符。

```Java
mv.visitInvokeDynamicInsn("compare", // 实现的方法名
                          "()Ljava/util/Comparator;", // 要求返回Comparator，注意泛型被擦除
                          new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false), // BSM的句柄
                          Type.getType("(Ljava/lang/Object;Ljava/lang/Object;)I"), // compare的方法描述符，泛型擦除到了Object
                          new Handle(H_INVOKESTATIC, "Test", "lambda$test$0", "(Ljava/lang/Number;Ljava/lang/Number;)I", false), // lambda实现方法的句柄
                          Type.getType("(Ljava/lang/Number;Ljava/lang/Number;)I") // 真正实现的方法描述符
);
mv.visitVarInsn(ASTORE, 2);
```

剩下的代码就比较简单了，就像下面这样：

```Java
mv.visitVarInsn(ALOAD, 0); // list
mv.visitVarInsn(ALOAD, 2); // comparator
mv.visitMethodInsn(INVOKEINTERFACE, "java/util/List", "sort", "(Ljava/util/Comparator;)V", true);
mv.visitVarInsn(ALOAD, 0); // list
mv.visitInsn(ICONST_0);
mv.visitMethodInsn(INVOKEINTERFACE, "java/util/List", "get", "(I)Ljava/lang/Object;", true); // 返回值被擦除到Object，这和List::get定义有关而和使用无关
mv.visitTypeInsn(CHECKCAST, "java/lang/Number"); // 强制转换到T的擦除类型Number，这是因为局部变量的类型是T
mv.visitVarInsn(ASTORE, 3);
mv.visitVarInsn(ALOAD, 3); // num
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Number", "intValue", "()I", false);
mv.visitVarInsn(ALOAD, 0); // list
mv.visitVarInsn(ALOAD, 1); // value
mv.visitVarInsn(ALOAD, 2); // comparator
mv.visitMethodInsn(INVOKESTATIC, "java/util/Collections", "binarySearch", "(Ljava/util/List;Ljava/lang/Object;Ljava/util/Comparator;)I", false);
mv.visitInsn(IADD);
mv.visitInsn(IRETURN);
mv.visitMaxs(4, 4);
```

这样我们的使用泛型的方法就构建好了。我们可以通过反射传入参数：

```Java
// Loader::defineClassNow是自定义的类加载器加载类的方法
Class<?> generated = new Loader().defineClassNow("Test", cw.toByteArray());
System.out.println(generated.getDeclaredMethod("test", List.class, Number.class)
	.invoke(null, new ArrayList<>(List.of(1, 2, 3, 4)), 3));
```

得到的结果是：

```SH
3
```

这就说明了我们的方法写入成功并成功地被JVM执行。

## 泛型的安全性

泛型的加入本质是为了确保代码的简洁和编译时辅助类型检查，也就是说，泛型能阻止类型的错误转换，就比如下面的例子：

```Java
List list = new ArrayList();
list.add("hello");
int i = (Integer) list.get(0);
```

这种代码在javac编译时是无法检查出语法和类型错误的：List的封装性造成了它内部数据的类型丢失，在get时只能得知对象是Object子类的对象但是不能得知确切的类型。但是在JVM运行时这就不一样了。JVM能知道任何对象的确切类型，因此在强制转换Integer时JVM能探测到`String->Integer`这种不可能的强制类型转换并抛出`ClassCastException`。

当泛型加入后，这种情况被改变了：

```Java
List<String> list = new ArrayList<>();
list.add("hello");
int i = (Integer) list.get(0);
```

这样的代码在javac就不能通过编译了：get的返回值是类型参数T（通过字节码中的泛型签名得知），在前面的声明中已经定义为String，因此javac可以探测到这个不可能的类型转换，抛出编译异常阻止编译，保证类型的安全性。

总而言之，**泛型确保了源码级别上的类型安全性**。

但是在字节码上看来，这就是另外一回事了。因为“泛型擦除”机制，字节码是不能使用泛型检查的，只能通过类型参数的已知超类约束泛型。但是这种约束不能阻止我们使用错误的类型传入：类验证时是无法检查泛型的。下面使用反射（本质和字节码差不多）举一个例子，使用到了我们刚才写的泛型方法：

```Java
System.out.println(generated.getDeclaredMethod("test", List.class, Number.class)
	.invoke(null, new ArrayList<>(List.of("1", "2", "3", "4")), 3));
```

这里是不会产生任何警告的（即使在IDE中），但是我们能清楚的看出它不符合我们对于test的定义：test要求传入的是`List<? extends T>`和`T`的形参，并且T要求是Number的子类，但是我们实际传入的是`List<String>`和`Number`，明显不符合test的形参列表。可是我们是能成功调用的：类型擦除让这些类型回到了原始类型，也就是说test的形参列表变成了`List`和`Number`，这时与我们传入的对象相符合，因此反射调用不会产生问题，但是在方法内的具体实现时会使用到checkcast用于类型转换，这里就会被JVM检测到异常并抛出`ClassCastException`。对于我们的这个调用，它最后的结果是：

```SH
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at io.github.nickid2018.asmtest.ASMMain.main(ASMMain.java:23)
Caused by: java.lang.ClassCastException: class java.lang.String cannot be cast to class java.lang.Number (java.lang.String and java.lang.Number are in module java.base of loader 'bootstrap')
	at java.base/java.util.TimSort.countRunAndMakeAscending(TimSort.java:355)
	at java.base/java.util.TimSort.sort(TimSort.java:220)
	at java.base/java.util.Arrays.sort(Arrays.java:1307)
	at java.base/java.util.ArrayList.sort(ArrayList.java:1721)
	at Test.test(Unknown Source)
	... 5 more
```

这样的结果说明了泛型在字节码中是不能保证安全的，类验证无法通过泛型签名阻止错误调用。因此在写入字节码时，遇到泛型必须严谨的检查，否则就会因为类型信息的丢失造成了类型的不安全行为，并且在写字节码时应尽量避免使用泛型。泛型的便利性仅体现在源码上，字节码中的泛型写入很麻烦。

---

有关于泛型的字节码知识就到这里了，有错误可以在评论指出。

到这里有关于具体方法字节码的写入就结束了，之后主要是有关于类结构的详解和ASM库各种工具的讲解。

下一篇文章暂定为类的结构（一）。