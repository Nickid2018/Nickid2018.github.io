---
title: Java ASM详解：模块
layout: default
date: 2022/10/17
category:
	- Java
tags:
 - Java
 - ASM
comments: true
toc: true
---

在前两篇专栏中我们讨论了各种类的结构，将各种类组织一起它们就组成了**模块**（Module）。

## 模块的定义

模块是在 Java 9 中引入的，它在`module-info.java`文件中定义。这个文件的内容是一个模块的描述，下面就是一个例子：

```java
open module JavaASMTest {
    requires org.objectweb.asm;
    exports io.github.nickid2018.asmtest to nickid2018.asm;
    uses java.sql.Driver;
    provides java.sql.Driver with io.github.nickid2018.asmtest.TestDriver;
}
```

上面的代码声明了一个模块，名称为`JavaASMTest`，并依赖模块`org.objectweb.asm`。

在模块中，我们可以定义以下信息：

1. **requires** 语句，它代表了这个模块需要依赖于什么其它的模块。所有的模块都隐式定义了`requires java.base`。

    requires 语句可以使用 **static** 修饰符，代表**静态依赖**。这种依赖在编译期是必须的，但是在运行时可以不存在。

	requires 语句也可以使用 **transitive** 修饰符，它代表了依赖此模块的模块将隐式获得对本模块依赖的模块的可读性。举一个例子：如果 A 依赖于 B，B 依赖于 C，那么 C 对 A 来说不可读，除非 A 显式定义 requires C；但是加上这个关键词，C 将对 A 可读。

2. **exports** 语句，它代表了模块内某个包内的 **public 成员**可以被其他模块访问，也就是开放了**可读性**。如果一个模块代码中直接访问了一个其他模块没有 exports 的成员，那么代码不通过编译。 exports 语句可以和 **to** 搭配，代表对指定模块开放这些成员。

3. **opens** 语句，和 exports 类似，但是它开放的是**访问性**。也就是说，允许通过反射访问模块中的类。这个语句也可以和 to 搭配。 当模块被定义为`open module`时，模块内的所有成员都将开放访问，并且不允许在模块中重新定义 opens。

4. **uses** 语句，用于使用服务。服务通常是抽象类，比如说`java.sql.Driver`。服务类所在的模块必须使用 requires 进行依赖。

5. **provides ... with ...** 语句，它代表提供服务的实现类。

下面是 opens 和 exports 搭配使用的情况：

| opens | exports |  通过类名直接访问  | 通过反射访问和执行 |
|:-----:|:-------:|:--------------:|:---------:|
| false |  false  |     编译错误     |   运行时异常   |
| true  |  false  |     编译错误     |   正常访问    |
| false |  true   |     正常访问     |   运行时异常   |
| true  |  true   |     正常访问     |   正常访问    |

## 模块的字节码表示

和普通的类一样，模块也是使用 ClassWriter 写入的，但是写入类信息的时候和普通的类不一样：它要求类名称是`module-info`，仅带有`ACC_MODULE`访问标志，并且没有超类和签名。

```java
cw.visit(V17, ACC_MODULE, "module-info", null, null, null);
```

接下来就是写入模块信息，这里用到了`visitModule`方法。如果模块是`open`的，那么访问标志要写为`ACC_OPEN`，否则写为0。最后一个参数是模块的版本，可以为`null`。

```java
ModuleVisitor mv = cw.visitModule("JavaASMTest", ACC_OPEN, null);
```

模块中有三种定义信息的模式：显式定义，也就是在文件中显式写出；隐式定义，由模块的依赖关系自行推断而出；既不是显式定义也不是隐式定义，由编译器编译时决定添加。如果访问标志含有`ACC_MANDATED`，则说明这个信息是被隐式定义的；如果访问标志含有`ACC_SYNTHETIC`，则这个模块信息既不是显式定义也不是隐式定义。所有使用访问标志的模块信息都可以使用这两个标志，下文将不再重复出现。

接下来就是填充模块的信息，这些信息与`ModuleVisitor`类中的方法一一对应。

**requires** 语句对应`visitRequire`方法，第一个参数是依赖模块的名称，最后一个参数是依赖模块的版本。第二个参数访问标志可以使用`ACC_TRANSITIVE`和`ACC_STATIC_PHASE`，分别代表了 transitive 和 static。

由于所有的模块都隐式定义了`requires java.base`，所以所有模块都必须像下面一样写入：

```java
mv.visitRequire("java.base", ACC_MANDATED, "17"); // 最后一个参数是对应 Java 版本
```

**exports** 和 **opens** 语句分别使用了`visitExport`和`visitOpen`方法，参数分别是全限定包名、访问标志和指定模块列表，最后一个参数可以为`null`，代表向所有模块开放。

```java
mv.visitExport("io/github/nickid2018/asmtest", 0, "nickid2018.asm");
```

**uses** 语句使用`visitUse`方法写入，它唯一的参数代表服务类全限定名。

```java
mv.visitUse("java/sql/Driver");
```

**provides ... with ...** 语句使用`visitProvide`方法，其中第一个参数代表服务类的全限定名，第二个则是实现类的列表。

```java
mv.visitProvide("java/sql/Driver", "io/github/nickid2018/asmtest/TestDriver");
```

现在，ModuleVisitor 类中还有两个方法我们没有说到：`visitPackage`和`visitMainClass`。这两个方法的实际作用不大，所以不讲解它的用法。

---

到这里模块的知识就结束了，下一篇专栏将讲述注解的使用。

若无特殊说明，所有字节码均以 Java 17 为标准。

参考资料：
* [The Java Language Specification (Java SE 17 Edition)](https://docs.oracle.com/javase/specs/jls/se17/html/index.html)
* [The Java Virtual Machine Specification (Java SE 17 Edition)](https://docs.oracle.com/javase/specs/jvms/se17/html/index.html)
* [ASM 9.4 Javadoc](https://asm.ow2.io/javadoc/index.html)