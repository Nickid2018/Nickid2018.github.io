---
title: Java ASM详解：类的结构（二）
layout: default
date: 2022/6/2
category:
    - Java
tags:
 - Java
 - ASM
comments: true
toc: true
---

上次专栏讲解了普通类的结构，这篇专栏将继续讲解接口、注解、枚举和记录这四种特殊的类。

## 接口

接口类似于抽象类，内部含有静态方法、公开抽象方法、公开默认实例方法、私有实例方法和常量字段。

声明一个接口，需要在`ClassWriter::visit`时同时使用`ACC_ABSTRACT`和`ACC_INTERFACE`访问标志。如果只含有`ACC_INTERFACE`标志，在加载这个类的时候 JVM 就会抛出`java.lang.ClassFormatError: Illegal class modifiers in class *: 0x200`的异常。接口的继承本质其实是实现，也就是说接口的父类仍然是`Object`，但是实现接口列表可以加入其它的接口。

如果接口是一个内部类，在使用时类似于静态内部类，也就是说内部接口不需要外部类实例作为依托。（但是在使用`visit`时不用写`ACC_STATIC`，这是一种等效）同样的，如果接口内部有内部类，那么内部类也等效于静态内部类。

紧接着说说接口的字段。接口内只能存在一种字段，那就是公开常量字段，也就是PSF（public static final）字段。在`visitField`时必须同时使用`ACC_PUBLIC`、`ACC_STATIC`和`ACC_FINAL`访问标志，否则在加载这个类的时候就会抛出`java.lang.ClassFormatError: Illegal field modifiers in class *: *`的异常。

接着来看看方法。接口不存在构造函数，并且只允许两种访问修饰符：`public`和`private`（引入于 Java 9），`protected`不能在接口中使用，不写访问修饰符则默认公开。

接口内可以定义静态方法，与普通的类没有太多差别，只是访问修饰有差别。对于接口内的实例方法，在 Java 层如果不定义`default`或者`private`那就会自动加上`abstract`，但是对于字节码来说，除了访问修饰外所有的定义都和普通的类一样。

接口也可以使用桥接方法。

下面是个例子，要生成下面的类：

```java
public interface Test {
    int TEST = 0; // PSF
    void test1(); // 抽象方法
    static void test2() {} // 公开静态方法
    default void test3() {} // 公开默认方法
    private void test4() {} // 私有方法(Java 9+)
    private static void test5() {} // 私有静态方法(Java 9+)
}
```

将使用下面的代码：

```java
// 省略MethodVisitor和FieldVisitor的使用
ClassWriter cw = new ClassWriter(0);
cw.visit(V17, ACC_INTERFACE + ACC_ABSTRACT, "Test", null, "java/lang/Object", null);
cw.visitField(ACC_PUBLIC + ACC_STATIC + ACC_FINAL, "TEST", "I", null, 0);
cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "test1", "()V", null, null);
cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "test2", "()V", null, null);
cw.visitMethod(ACC_PUBLIC, "test3", "()V", null, null);
cw.visitMethod(ACC_PRIVATE, "test4", "()V", null, null);
cw.visitMethod(ACC_PRIVATE + ACC_STATIC, "test5", "()V", null, null);
cw.visitEnd();
```

## 注解

注解类型是特殊的接口，除了接口都具有的特性之外还增加了一些限制。

先说一下声明。注解类型除了接口要求的两个访问标志外还需要添加一个`ACC_ANNOTATION`标志，且必须只实现（或者说是 Java 层的继承）于`java.lang.annotation.Annotation`。（注意：JVM 对实现`Annotation`接口这件事不加以检查，但是如果你使用了反射尝试获取这个注解时会报错）

注解对于方法的要求很严格：要求不能有私有方法、默认方法和静态方法，只能存在公开抽象方法。

到这里你可能会问：注解方法的默认值是怎么实现的？其实默认值不是方法体，而是使用了`visitAnnotationDefault`这个方法用于写入默认值。

下面我们要生成这个注解：

```java
public @interface Test {
    String test1();
    String test2() default "";
}
```

可以使用下面的代码生成：

```java
ClassWriter cw = new ClassWriter(0);
cw.visit(V17, ACC_PUBLIC + ACC_INTERFACE + ACC_ABSTRACT + ACC_ANNOTATION, "Test", null, "java/lang/Object", new String[]{"java/lang/annotation/Annotation"});
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "test1", "()Ljava/lang/String;", null, null);
mv.visitEnd();
mv = cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "test2", "()Ljava/lang/String;", null, null);
AnnotationVisitor av = mv.visitAnnotationDefault(); // 写入默认值
av.visit(null, ""); // 这里以后会说，name写入null是要求
av.visitEnd();
mv.visitEnd();
cw.visitEnd();
```

## 枚举

枚举是一种特殊的类，主要用于存贮常量。和普通的类相比，它有下面的性质：
* 所有枚举都继承于`java.lang.Enum`，并且都为`final`。
* 枚举的所有构造函数都是私有的。
* 自动生成`values`和`valueOf`方法。
* 作为内部类时等效静态。

写入一个枚举，在`visit`时要将`ACC_ENUM`和`ACC_FINAL`访问标志同时写入，并且父类必须写为`java/lang/Enum`。由于`Enum`带有泛型，所以`signature`也要写入。例如，下面这个枚举：

```java
public enum Test {
    // 省略
}
```

在声明时必须用下面的代码：

```java
cw.visit(V17, ACC_ENUM + ACC_FINAL + ACC_PUBLIC, "Test",
        "Ljava/lang/Enum<LTest;>;",  // 泛型签名
        "java/lang/Enum", null);
```

在使用 Java 编写枚举类时可以写两种字段：一种是普通的字段，这个和普通的类一样，没有限制；另一种就是枚举字段，它必须是枚举类的对象，并且是 PSF 字段还带有`ACC_ENUM`访问标志。例如下方的枚举字段 A：

```java
public enum Test {
    A
}
```

在声明时应该遵照下面的方式：

```java
// 只保留字段定义
cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC + ACC_ENUM, "A", "LTest;", null, null);
```

在字节码中，除了上面的两种字段外，枚举中还有一个字段是系统生成用于保存所有枚举字段的私有常量，`$VALUES`。它的访问标志除了`ACC_PRIVATE`、`ACC_STATIC`和`ACC_FINAL`外还带有`ACC_SYNTHETIC`，这代表它是编译时自动生成的。它的类型是这个类的数组。

`$VALUES`存在的价值是为了`values`方法和`Enum`的索引。在介绍它的用途之前先来说说`$VALUES`和枚举字段的初始化。

枚举字段和普通常量的初始化一样，也是简单的创建、调用构造函数和赋值。但是不同的是，枚举的构造函数和普通的构造函数不同，它默认带有两个形参。实际上，枚举的默认构造函数是这样的：

```java
private Test(String name, int ordinal) {
    super(name, ordinal);
}
```

枚举默认构造函数的写入如下：

```java
// 省略ClassWriter创建和visit
MethodVisitor mv = cw.visitMethod(ACC_PRIVATE, "<init>", "(Ljava/lang/String;I)V", null, null);
mv.visitVarInsn(ALOAD, 0); // this
mv.visitVarInsn(ALOAD, 1);
mv.visitVarInsn(ILOAD, 2);
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Enum", "<init>", "(Ljava/lang/String;I)V", false);
mv.visitInsn(RETURN);
mv.visitMaxs(3, 3);
mv.visitEnd();
```

如果枚举类有定义构造函数，那么在字节码中仍然需要将这两个形参添加到 Java 源代码的形参列表之前，且必须调用`Enum`的这个父类构造函数。

所以枚举字段的创建对于上面的 A 来说就像下面这样：

```java
// 省略mv创建，mv现在是<clinit>的visitor
mv.visitTypeInsn(NEW, "Test");
mv.visitInsn(DUP);
mv.visitLdc("A"); // 枚举字段的名称
mv.visitInsn(ICONST_0); // A是第一个枚举常量字段，编号是0
mv.visitMethodInsn(INVOKESPECIAL, "Test", "<init>", "(Ljava/lang/String;I)V", false);
mv.visitFieldInsn(PUTSTATIC, "Test", "A", "LTest;");
```

`$VALUES`的赋值不太一样，它是委托到了另一个方法`$values`生成的。这个方法带有`ACC_PRIVATE`、`ACC_STATIC`和`ACC_SYNTHETIC`访问标志，且方法返回值是该类的数组，形参列表为空。它的作用就是创建一个数组，并将所有枚举字段按顺序存储进数组之中并返回。对于上面的 Test 就是这样的：

```java
// 省略ClassWriter创建
MethodVisitor mv = cw.visitMethod(ACC_PRIVATE + ACC_STATIC + ACC_SYNTHETIC, "$values", "()[LTest;", null, null);
mv.visitInsn(ICONST_1);
mv.visitTypeInsn(ANEWARRAY, "Test"); // 创建长度为1的数组（枚举字段的数量）
mv.visitInsn(DUP);
mv.visitInsn(ICONST_0);
mv.visitFieldInsn(GETSTATIC, "Test", "A", "LTest;");
mv.visitInsn(AASTORE); // 放入A
mv.visitInsn(ARETURN);
mv.visitMaxs(4, 0);
mv.visitEnd();
```

在静态初始化中，`$VALUES`直接由`$values`的返回值赋值：

```java
// mv创建省略，此时为<clinit>
mv.visitMethodInsn(INVOKESTATIC, "Test", "$values", "()[Test;", false);
mv.visitFieldInsn(PUTSTATIC, "Test", "$VALUES", "[LTest;");
```

我们用到的`values`是另一个方法，也是由编译器自动生成的。它返回的是`$VALUES`的副本，Java 的代码像这样：

```java
public static Test[] values() {
    return (Test[]) $VALUES.clone();
}
```

> 可以注意到这个方法不存在`try-catch`，即使`clone`定义了抛出`CloneNotSupportedException`。JVM对这种异常处理不检查，可以说在字节码范围内，异常处理是可有可无的。

字节码像这样：

```java
// 省略ClassWriter创建
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "values", "()[LTest;", null, null);
mv.visitFieldInsn(GETSTATIC, "Test", "$VALUES", "[LTest;");
mv.visitMethodInsn(INVOKEVIRTUAL, "[LTest;", "clone", "()Ljava/lang/Object;", false);
mv.visitTypeInsn(CHECKCAST, "[LTest;");
mv.visitInsn(ARETURN);
mv.visitMaxs(1, 0);
mv.visitEnd();
```

另一个会自动创建的方法，`valueOf`，用 Java 表示是这样的：

```java
public static Test valueOf(String name) {
    return (Test) Enum.valueOf(Test.class, name);
}
```

字节码写入如下：

```java
// 省略ClassWriter创建
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "valueOf", "(Ljava/lang/String;)LTest;", null, null);
mv.visitLdcInsn(Type.getType("LTest;")); // Test.class
mv.visitVarInsn(ALOAD, 0);
mv.visitMethodInsn(INVOKESTATIC, "java/lang/Enum", "valueOf", "(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;", false);
mv.visitTypeInsn(CHECKCAST, "Test");
mv.visitInsn(ARETURN);
mv.visitMaxs(2, 1);
mv.visitEnd();
```

到这里一个完整的枚举才写完。可以看到我们必须写出`$VALUES`、`$values`、`values`、`valueOf`这些字段和方法，非常的麻烦。即使一个非常简单的枚举也必须有所有这些要素，所以枚举的写入很繁琐，还要注意别忘了它的组件。

## 记录

记录也是一种特殊的类，它在 Java 14 开始加入。和普通的类相比，它有下面的不同之处：

* 不能单独定义实例字段，所有终态实例字段都要在类之后的括号定义。
* 继承于`java.lang.Record`，且都为`final`。
* 自动生成`toString`、`hashCode`和`equals`。（除非自行定义）
* 作为内部类时等效静态。

写入一个记录，在`visit`时必须带有`ACC_FINAL`和`ACC_RECORD`访问标志，并且要继承`java/lang/Record`。接下来以下面的记录作为例子：

```java
public record Test(int a) {
}
```

在写入时要这样定义：

```java
// 省略了ClassWriter创建
cw.visit(V17, ACC_PUBLIC + ACC_FINAL + ACC_SUPER + ACC_RECORD, "Test", null, "java/lang/Record", null);
```

对于记录的终态实例字段（也可以叫记录字段），它只能含有`ACC_PRIVATE`和`ACC_FINAL`这两个访问标志。它需要两次定义：一次是普通的字段定义，使用`visitField`；另一次是记录组件的定义，使用`visitRecordComponent`，在定义字段之前写入。这里的`a`就像下面这样定义：

```java
// 省略ClassWriter创建和Visitor使用
cw.visitRecordComponent("a", "I", null); // 没有泛型所以最后一项是null
cw.visitField(ACC_PRIVATE + ACC_FINAL, "a", "I", null, null);
```

每个记录字段都有自动生成的对应的 Getter，代码很简单，就像下面这样：

```java
public int a() {
    return a;
}
```

字节码像这样写：

```java
// 省略ClassWriter创建
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "a", "()I", null, null);
mv.visitVarInsn(ALOAD, 0); // this
mv.visitFieldInsn(GETFIELD, "Test", "a", "I");
mv.visitInsn(IRETURN);
mv.visitMaxs(1, 1);
mv.visitEnd();
```

记录的默认构造函数和记录字段有关，形参列表正好和记录字段的顺序一致。对于上面的`Test`，构造函数是这样的：

```java
public Test(int a) {
    super();
    this.a = a;
}
```

转换成字节码如下：

```java
// 省略ClassWriter创建
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "(I)V", null, null);
mv.visitVarInsn(ALOAD, 0);
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Record", "<init>", "()V", false);
mv.visitVarInsn(ALOAD, 0);
mv.visitVarInsn(ILOAD, 1);
mv.visitFieldInsn(PUTFIELD, "Test", "a", "I");
mv.visitInsn(RETURN);
mv.visitMaxs(2, 2);
mv.visitEnd();
```

记录必须有`toString`、`hashCode`、`equals`这三个方法，这是因为在`Record`中声明了它们3个是抽象的。如果我们不自己写这三个方法，那么系统在编译的时候会自动生成。

自动生成的这三个方法都用到了`invokedynamic`，使用的引导方法都是`java.lang.runtime.ObjectMethods.bootstrap`。这个方法的定义是：

```java

public static Object bootstrap(
                MethodHandles.Lookup lookup, // 引导方法必要参数
                String methodName,           // 引导方法必要参数，实现的方法名称
                TypeDescriptor type,         // 引导方法必要参数，实现的方法描述符
                Class<?> recordClass,        // 记录的Class<?>对象
                String names,                // 记录实例字段的名称序列
                MethodHandle... getters      // 获取记录实例字段的句柄
) throws Throwable
```

其中`names`是记录实例字段的名称序列，用分号`;`隔开。

下面仅给出`toString`的代码，另两个除了`methodName`和`type`不同外没有差别。

```java
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC + ACC_FINAL, "toString", "()Ljava/lang/String;", null, null);
mv.visitVarInsn(ALOAD, 0); // this
mv.visitInvokeDynamicInsn("toString",  // 方法名
        "(LTest;)Ljava/lang/String;", // 方法描述符
        new Handle(H_INVOKESTATIC, "java/lang/runtime/ObjectMethods", "bootstrap", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/TypeDescriptor;Ljava/lang/Class;Ljava/lang/String;[Ljava/lang/invoke/MethodHandle;)Ljava/lang/Object;", false),  // 引导方法
        new Object[]{
            Type.getType("Lio/github/nickid2018/asmtest/ASMMain$A;"), // 记录类对象
            "a",  // 实例字段名称序列
            new Handle(H_GETFIELD, "Test", "a", "I", false) // 字段a的句柄
        });
mv.visitInsn(ARETURN);
mv.visitMaxs(1, 1);
mv.visitEnd();
```

到这里记录才算写入完毕。

---

到这里类的结构就结束了，接下来的文章将讨论好玩的东西（因为还没想出来）。

这系列专栏没有特殊声明都是 Java 17 的字节码，请注意使用。