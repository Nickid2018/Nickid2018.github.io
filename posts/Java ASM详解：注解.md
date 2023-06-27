---
title: Java ASM详解：注解
layout: default
date: 2022/10/21
category:
 - Java
tags:
 - Java
 - ASM
comments: true
toc: true
---

在 Java 语言中，**注解**（Annotation）是很重要的一部分。它的存在让许多代码变得简洁。与**注解执行器**（Annotation
Processor）结合之后，它能发挥出意想不到的功能。这篇文章就将讲述注解是什么写入在字节码中的。

> 注解类型的定义在前文（类的结构二）中已经写过了，如果不了解注解类型的写入可以先查看那一篇文章。

## 写入注解信息

写入注解信息需要用到`AnnotationVisitor`这个类，它含有的方法支持我们对注解添加信息。

### 写入常量

写入常量需要使用`visit`方法，它的第一个参数代表注解属性的名称，第二个参数代表值。可以写入的常量类型有：8种基本类型、字符串和
Type 对象（不包括方法描述符）。

### 写入枚举常量

写入枚举常量需要使用`visitEnum`方法，第一个参数仍然是注解属性的名称，第二个是枚举对象的**类型描述符**，第三个是枚举对象的名称。

### 写入其他注解类型

写入其他注解类型对象需要使用`visitAnnotation`方法，第一个参数是注解属性名称，第二个是要写入的注解类型的**类型描述符**
。这个方法返回一个新的`AnnotationVisitor`，使用这个新的 visitor 可以填充要写入的注解类型对象的信息。

假如说我们要写入下面的注解：

```java
// 省略 Target 等描述注释
@interface TestAnno {
    String testStr();
    Class<?> testClass();
}

@interface Test {
    TestEnum testEnum();
    TestAnno testAnnotation();
}

@Test(testEnum = TestEnum.A, testAnnotation = @TestAnno(testStr = "Test", testClass = Test.class))
```

我们需要写下面这些代码：

```java
// 已省略 Test 的 AnnotationVisitor 的创建，名称为av
av.visitEnum("testEnum", "LTestEnum;", "A"); // 写入 testEnum
AnnotationVisitor av1 = av.visitAnnotation("testAnnotation", "LTestAnno;"); // 写入 testAnnotation
av1.visit("testStr", "Test"); // 写入 testStr
av1.visit("testClass", Type.of("LTest;")); // 写入 testClass
av1.visitEnd();
```

### 写入数组

除了上述类型之外，注解类型还允许在注解中定义一维数组。写入数组需要用到`visitArray`
方法，参数为注解属性名称，返回一个新的`AnnotationVisitor`用于填充这个数组。写入数组信息时，所有的注解属性名称都要写为`null`
，并且不允许再调用`visitArray`，因为注解类型不允许二维及多维数组的存在。

下面是一个例子：

```java
// 省略 Target 等描述注释
@interface Test {
    String[] value();
}

@Test({"test1", "test2"})
```

写入需要下面的代码：

```java
// 已省略 Test 的 AnnotationVisitor 的创建，名称为av
AnnotationVisitor av1 = av.visitArray("value");
av1.visit(null, "test1");
av1.visit(null, "test2");
av1.visitEnd();
```

### 写入带有@Repeatable注解的注解类型

通常情况下，一个注解位置每个注解类型只能声明一次，但是带有`@Repeatable`的注解类型可以多次声明。假设有下面的注解类型：

```java
// 省略 Target 等描述注释
@Repeatable(TestContainer.class)
@interface Test {
    String value();
}

@interface TestContainer {
    Test[] value();
}
```

如果在注释位置上只有一个`Test`注解，那么写入时只需要写`Test`；但是如果一个位置上有多个`Test`注解，则应该使用`TestContainer`
进行等效代替并写入，像下面这样：

```java
@Test("hello")
@Test("world")
// 等价于 ...
@TestContainer({@Test("hello"), @Test("world")})
```

## 注解类型的可见性

根据不同注解类型的作用，在定义注解类型时我们通常都会设置它的可见性，也就是使用`@Retention`进行设置。

| RetentionPolicy | SOURCE | CLASS | RUNTIME |
|:---------------:|:------:|:-----:|:-------:|
|      写入类文件      | false  | true |  true   |
|    可见性          |   -    | false | true |

可见性影响了反射时我们能不能访问到这个注解，如果不写则默认为`false`。

## 注解的写入位置

注解不是哪里都能写入的，它有一套非常详细的使用方法，下面我们将分类讲解。

### 类和类成员定义时的注解

当类与类的成员（字段、方法、记录元素）被定义时，它可以附加注解，例如：

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {
    @Deprecated
    String value();
}
```

这种注解在写入时需要调用各个 visitor 的`visitAnnotation`方法。其中第一个参数是注解类型的**类型描述符**
，第二个是注解类型的**可见性**。

对于附加在类`TestAnnotation`上的注解`Retention`，我们需要这样写入：

```java
// 省略 ClassWriter 的创建，名称为 cw
AnnotationVisitor av = cw.visitAnnotation("Ljava/lang/annotation/Retention;", true);
av.visitEnum("value", "Ljava/lang/annotation/RetentionPolicy;", "RUNTIME");
av.visitEnd();
```

如果一个成员在被定义时被添加上了`@Deprecated`注解，那么在定义这个成员时也要加上`ACC_DEPRECATED`访问标志。

对于`value`方法，我们需要这样写入：

```java
// 省略 ClassWriter 的创建，名称为 cw
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT + ACC_DEPRECATED, "value", "()Ljava/lang/String;", null, null);
AnnotationVisitor av = mv.visitAnnotation("Ljava/lang/Deprecated;", true);
av.visitEnd();
mv.visitEnd();
```

### 类型的注解

在类型上，我们也能加入注解，这些注解`@Target`中必须含有`TYPE_USE`。要知道类型的注解怎么插入，我们需要先了解两个概念：**类型路径**（Type Path）和**类型引用**（Type
Reference）。

#### 类型路径

为了指定注解在类型中的位置，JVM 引入了类型路径。假设我们有下面一个泛型需要进行注解：

```java
@A Map<@B?extends @C String, @D List<@E Object>>
@A Outer.@B Middle.@C Inner
@A String @B[] @C[] @D[]
```

可以看到，注解 ABCDE
分别注解了一个复杂类型中的不同元素。如果我们从类型的最外层开始对类型的参数进行遍历，我们就能最终指定插入的位置。遍历的每一**
步**（step）都含有两个属性：

* **类型路径类别**（Type Path Kind），它决定了这一步该走向哪种元素，可以使用的值如下表：

|       类别       | 对应元素     | 举例                        |
|:--------------:|:---------|:--------------------------|
| ARRAY_ELEMENT  | 数组类型     | `@A Test[]`               |
|   INNER_TYPE   | 非静态嵌套类   | `Outer.@A Inner`          |
| WILDCARD_BOUND | 限定通配符的类型 | `Test<? extends @A Type>` |
| TYPE_ARGUMENT  | 泛型参数     | `Map<@A Test, Object>`    |

* **类型参数索引**（Type Argument Index），代表在同一级的几个相同类别的元素中要选择哪个。例如在`Map<@A Test, Object>`
  中，泛型参数共有两个，要指定其中一个需要用到这个索引。请注意，只有**泛型参数**才需要指定索引，其它的类别不需要。

假设我们有下面的类型需要注解：

```java
Map<String, List<@A Object>>
```

那么我们访问到注解 A 的路径就像这样：

```java
Map<String, List<@A Object>>
  - List<@A Object> // kind = TYPE_ARGUMENT, index = 1
    - @A Object // kind = TYPE_ARGUMENT, index = 0
```

对于数组来说，注解的位置影响了它的路径长度，下面是按照 A 注解类型路径长度逐渐增大排序的注解示例：

```java
String @A[][][] // 路径长度0
String []@A[][] // 路径长度1
String [][]@A[] // 路径长度2
@A String[][][] // 路径长度3
```

下面是一个组合的例子：

```java
// 假设 Inner 是 Outer 的非静态内部类
List<? super Outer.Inner<@A String[][]>>
  - ? super Outer.Inner<@A String[][]> // kind = TYPE_ARGUMENT, index = 0
    - Outer.Inner<@A String[][]> // kind = WILDCARD_BOUND
      - Inner<@A String[][]> // kind = INNER_TYPE
        - @A String[][] // kind = TYPE_ARGUMENT, index = 0
          - @A // kind = ARRAY_ELEMENT, path_length = 2
```

在 ASM 库中，类型路径使用`TypePath`包装，创建一个 TypePath 对象需要使用`fromString`方法，它的参数是一个字符串，这个字符串里面存储了可以复原
TypePath 的所有信息。对于每一步，都有一个对应关系，这些步按顺序连接起来就能复原 TypePath。

|       类别       | 映射字符串      |
|:--------------:|:-----------|
| ARRAY_ELEMENT  | `[`        |
|   INNER_TYPE   | `.`        |
| WILDCARD_BOUND | `*`        |
| TYPE_ARGUMENT  | `<index>;` |

上面的例子可以用`0;*.0;[[`代替。

#### 类型引用

类型路径决定了注解在一个类型内的位置，而类型引用指定了这个被注解类型的位置。

类型引用本质上是一个 int，其中第25~32位是引用的类型，1~24位是引用的参数。ASM 库提供了`TypeReference`类来简化创建这些数字的代码。

先来说说无参的类型引用类型，这些类型可以使用`newTypeReference`创建 TypeReference 对象，之后通过`getValue`方法获得 int
形式的类型引用，如下表：

|          类型           |                     传入方法                      | 说明               |
|:---------------------:|:---------------------------------------------:|:-----------------|
|         FIELD         |      `FieldVisitor::visitTypeAnnotation`      | 使用在字段声明的类型上      |
|     METHOD_RETURN     |     `MethodVisitor::visitTypeAnnotation`      | 使用在方法声明的返回值上     |
|    METHOD_RECEIVER    |     `MethodVisitor::visitTypeAnnotation`      | 使用在方法接收器上[1]     |
|    LOCAL_VARIABLE     | `MethodVisitor::visitLocalVariableAnnotation` | 使用在方法局部变量上       |
|   RESOURCE_VARIABLE   | `MethodVisitor::visitLocalVariableAnnotation` | 使用在方法资源变量上[2]    |
|      INSTANCEOF       |     `MethodVisitor::visitInsnAnnotation`      | 使用在`instanceof`后 |
|          NEW          |     `MethodVisitor::visitInsnAnnotation`      | 使用在`new`后        |
| CONSTRUCTOR_REFERENCE |     `MethodVisitor::visitInsnAnnotation`      | 使用在构造函数引用后[3]    |
|   METHOD_REFERENCE    |     `MethodVisitor::visitInsnAnnotation`      | 使用在方法引用后[3]      |

> [1]：方法接收器（`this`）可以被**显式定义**，比如非静态方法`Test::test()`在定义时可以写为`test(Test this)`，这时可以在 this 上附加注解。
> [2]：资源局部变量是使用`try-with-resource`语法产生的局部变量。
> [3]：构造函数引用属于方法引用，语法为`<ClassName>::new`。这两个注解必须在`invokedynamic`字节码后。

接下来说一下有参的类型引用类型。

* 需要**类型参数**的类型引用。它们需要使用`newTypeParameterReference`获得 TypeReference 对象，第二个参数是类型参数的序号。

|           类型           |                                          传入方法                                          | 说明         |
|:----------------------:|:--------------------------------------------------------------------------------------:|:-----------|
|  CLASS_TYPE_PARAMETER  | `ClassVisitor::visitTypeAnnotation` <br> `RecordComponentVisitor::visitTypeAnnotation` | 使用在类类型参数上  |
| METHOD_TYPE_PARAMETER  |                            `MethodVisitor::visitTypeAnnotation`                        | 使用在方法类型参数上 |

* 需要**类型参数边界**的类型引用。类型参数边界即`<T extends ...>`
  这种类型参数后面的限定，可以不止一个。它们需要使用`newTypeParameterBoundReference`获得 TypeReference
  对象，第二个参数是类型参数的序号，第三个参数是规定边界限定的序号。

|              类型              |                                          传入方法                                          | 说明        |
|:----------------------------:|:--------------------------------------------------------------------------------------:|:----------|
|  CLASS_TYPE_PARAMETER_BOUND  | `ClassVisitor::visitTypeAnnotation` <br> `RecordComponentVisitor::visitTypeAnnotation` | 类类型参数边界   |
| METHOD_TYPE_PARAMETER_BOUND  |                                `MethodVisitor::visitTypeAnnotation`                    | 方法类型参数边界  |

* 需要**超类序号**
  的类型引用。超类序号是定义类时指定的继承类和实现类的序号，继承类的序号是-1，实现类的序号按照定义顺序从0计数。使用`newSuperTypeReference`
  创建 TypeReference 对象，第二个参数就是超类序号。类型固定为`CLASS_EXTENDS`，可用在`ClassVisitor::visitTypeAnnotation`
  和`RecordComponentVisitor::visitTypeAnnotation`方法中。
* 需要**方法形式参数序号**的类型引用。使用`newFormalParameterReference` 创建对象，类型固定为`METHOD_FORMAL_PARAMETER`
  ，可用在`MethodVisitor::visitTypeAnnotation`中。
* 需要**方法异常列表序号**的类型引用。使用`newExceptionReference`创建 TypeReference，类型固定为`THROWS`
  ，可用在`MethodVisitor::visitTypeAnnotation`中。
* 需要 **try-catch 块序号**的类型引用。用`newTryCatchReference`创建，类型是`EXCEPTION_PARAMETER`
  ，使用`MethodVisitor::visitTryCatchAnnotation`写入字节码。
* 需要**实际参数序号**的类型引用。它们需要使用`newTypeArgumentReference`
  创建，都需要使用`MethodVisitor::visitInsnAnnotation`写入。

|                  类型                  | 说明                      |
|:------------------------------------:|:------------------------|
|                 CAST                 | 使用在`checkcast`后         |
| CONSTRUCTOR_INVOCATION_TYPE_ARGUMENT | 使用在调用构造函数时指定的类型参数中[1]   |
|   METHOD_INVOCATION_TYPE_ARGUMENT    | 使用在调用方法时指定的类型参数中[1]     |
| CONSTRUCTOR_REFERENCE_TYPE_ARGUMENT  | 使用在使用构造函数引用时指定的类型参数中[2] |
|    METHOD_REFERENCE_TYPE_ARGUMENT    | 使用在使用方法引用时指定的类型参数中[2]   |

> [1]：在方法或者构造函数调用时，可以手动指定类型参数，如`Lists.<String>newArrayList()`。注解可以附加到类型参数的列表之中。
> [2]：方法引用也可以指定类型参数，如`Lists::<String>newArrayList`。注解可以附加到这个类型参数列表中。这两个注解必须在`invokedynamic`字节码后。

下面是一个例子：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.TYPE_USE})
@interface Test {
    String value();
}

public abstract class TestMain {
    public abstract InputStream testFile(String test) throws IOException;

    @SuppressWarnings("unchecked")
    public <@Test("test") T extends @Test("testA") InputStream> void test(
            @Test("testB") TestMain this,
            @Test("testC") String file,
            Consumer<BiConsumer<InputStream, Integer>> consumer)
            throws @Test("testD") IOException {
        try (@Test("testE") T input = (@Test("testF") T) testFile(file)) {
            consumer.accept(@TestA("testG") InputStream::mark);
        }
    }
}
```

下面将写入这个 test 方法：

```java
// 省略 ClassWriter 创建，对象为 cw，使用 COMPUTE_FRAMES 标志
// 所有注解具体写入内容省略，仅显示目前写入哪个注解
// 省略 SuppressWarnings 注解
MethodVisitor mv = classWriter.visitMethod(ACC_PUBLIC, "test",
        "(Ljava/lang/String;Ljava/util/function/Consumer;)V",
        "<T:Ljava/io/InputStream;>(Ljava/lang/String;Ljava/util/function/Consumer<Ljava/util/function/BiConsumer<Ljava/io/InputStream;Ljava/lang/Integer;>;>;)V",
        new String[]{"java/io/IOException"});
AnnotationVisitor av = mv.visitTypeAnnotation(newTypeParameterReference(METHOD_TYPE_PARAMETER, 0).getValue(), null, "LTest;", true);
// test
av = mv.visitTypeAnnotation(newTypeParameterBoundReference(METHOD_TYPE_PARAMETER_BOUND, 0, 0).getValue(), null, "LTest;", true);
// testA
av = mv.visitTypeAnnotation(newExceptionReference(0).getValue(), null, "LTest;", true);
// testD
av = mv.visitTypeAnnotation(newTypeReference(METHOD_RECEIVER).getValue(), null, "LTest;", true);
// testB
av = mv.visitTypeAnnotation(newFormalParameterReference(0).getValue(), null, "LTest;", true);
// testC
mv.visitCode();

// try-with-resource ---------------------
Label label0 = new Label(), label1 = new Label(), label2 = new Label();
mv.visitTryCatchBlock(label0, label1, label2, "java/lang/Throwable");
Label label3 = new Label(), label4 = new Label(), label5 = new Label();
mv.visitTryCatchBlock(label3, label4, label5, "java/lang/Throwable");

Label start = new Label();
mv.visitLabel(label6);
mv.visitVarInsn(ALOAD, 0); // this
mv.visitVarInsn(ALOAD, 1); // file
mv.visitMethodInsn(INVOKEVIRTUAL, "TestMain", "testFile", "(Ljava/lang/String;)Ljava/io/InputStream;", false);
mv.visitVarInsn(ASTORE, 3); // input
        
av = mv.visitInsnAnnotation(newTypeArgumentReference(CAST, 0).getValue(), null, "LTest;", true);
// testF
        
mv.visitLabel(label0);
mv.visitVarInsn(ALOAD, 2); // consumer
mv.visitInvokeDynamicInsn("accept",
        "()Ljava/util/function/BiConsumer;", 
        new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false),
        new Object[]{
                Type.getType("(Ljava/lang/Object;Ljava/lang/Object;)V"),
                new Handle(H_INVOKEVIRTUAL, "java/io/InputStream", "mark", "(I)V", false), 
                Type.getType("(Ljava/io/InputStream;Ljava/lang/Integer;)V")
        }
);

av = mv.visitInsnAnnotation(newTypeReference(METHOD_REFERENCE, 0).getValue(), null, "LTest;", true);
// testG
        
mv.visitMethodInsn(INVOKEINTERFACE, "java/util/function/Consumer", "accept", "(Ljava/lang/Object;)V", true);

// try-with-resource closing
mv.visitLabel(label1);
mv.visitVarInsn(ALOAD, 3);
Label label7 = new Label();
mv.visitJumpInsn(IFNULL, label7);
mv.visitVarInsn(ALOAD, 3);
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/InputStream", "close", "()V", false);
mv.visitJumpInsn(GOTO, label7);

mv.visitLabel(label2);
mv.visitVarInsn(ASTORE, 4); // (exception)
mv.visitVarInsn(ALOAD, 3); // input

Label label8 = new Label();
mv.visitJumpInsn(IFNULL, label8);
mv.visitLabel(label3);
mv.visitVarInsn(ALOAD, 3); // input
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/InputStream", "close", "()V", false);
mv.visitLabel(label4);
mv.visitJumpInsn(GOTO, label8);

mv.visitLabel(label5);
mv.visitVarInsn(ASTORE, 5); // (exception 2)
mv.visitVarInsn(ALOAD, 4); // (exception)
mv.visitVarInsn(ALOAD, 5); // (exception 2)
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Throwable", "addSuppressed", "(Ljava/lang/Throwable;)V", false);
 
mv.visitLabel(label8);
mv.visitVarInsn(ALOAD, 4); // (exception)
mv.visitInsn(ATHROW);

// try-with-resource end
mv.visitLabel(label7);
mv.visitLineNumber(49, label7);
mv.visitInsn(RETURN);

// LVT
Label label9 = new Label();
mv.visitLabel(label9);
mv.visitLocalVariable("input", "Ljava/io/InputStream;", "TT;", label0, label7, 3);
mv.visitLocalVariable("this", "LTestMain;", null, start, label9, 0);
mv.visitLocalVariable("file", "Ljava/lang/String;", null, start, label9, 1);
mv.visitLocalVariable("consumer", "Ljava/util/function/Consumer;", "Ljava/util/function/Consumer<Ljava/util/function/BiConsumer<Ljava/io/InputStream;Ljava/lang/Integer;>;>;", label6, label9, 2);

av = methodVisitor.visitLocalVariableAnnotation(newTypeReference(RESOURCE_VARIABLE).getValue(), null, new Label[]{label0}, new Label[]{label7}, new int[]{3}, "Lio/github/nickid2018/asmtest/ASMMain$TestA;", true);
// testE

mv.visitMaxs(2, 6);
mv.visitEnd();
```

### 方法形式参数上的注解

等下，我们刚刚不是说过形式参数上怎么插入注解了吗？事实上，形式参数可以使用两种注解：
一种是标记为`TYPE_USE`的类型注解，另一种是标记为`PARAMETER`的形式参数注解。如果一个注解同时拥有这两个标志，就都要写入字节码。

写入形式参数注解需要两步：写入注解形式参数数量、写入形式参数注解。

写入注解形式参数数量需要使用`visitAnnotableParameterCount`方法。假设我们有一个方法：

```java
// 仍然使用上一个例子中的类型
public void test(TestMain this, String s, @Test("test") int i)
```

它的形式参数是两个，因为**接收器 this 不是形式参数**。写入如下：

```java
mv.visitAnnotableParameterCount(2, true);
```

第二个参数代表了注解的**可见性**。

写入形式参数注解需要使用`visitParameterAnnotation`，参数类似定义注解的使用方法。

请注意：如果形式参数列表中同时存在运行时可见和不可见的注解，那么先写`visitAnnotableParameterCount`，可见性为`true`，在后面写出所有运行时可见的注解；之后再一行`visitAnnotableParameterCount`，可见性为`false`，在后面写出所有运行时不可见的注解。`visitAnnotableParameterCount`对于每个可见性只出现一次。

### 注解类型中的 default 默认值

在类的结构二中，我们说到了有默认值的注解属性怎么写入。它使用的是`visitAnnotationDefault`方法。

它返回的`AnnotationVisitor`需要写入一个`name`为`null`的属性，这个属性写入什么值和怎么写入取决于你要决定的默认值。

---

到这里有关于注解的相关知识都已经说完了，下篇专栏可能是 ASM Tree API 部分。