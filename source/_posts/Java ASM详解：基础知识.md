---
title: Java ASM详解： 基础知识
layout: default
date: 2020/7/23
updated: 2021/8/29
category:
	- Java
tags:
 - Java
 - ASM
comments: true
toc: true
---

## ASM是什么，字节码又是什么

>ASM是一个Java字节码分析、创建和修改的开源应用框架。它可以动态生成二进制格式的stub类或其他代理类，或者在类被Java虚拟机装入内存之前，动态修改类。在ASM中提供了诸多的API用于对类的内容进行字节码操作的方法。与传统的BCEL和SERL不同，在ASM中提供了更为优雅和灵活的操作字节码的方式

这是ASM官网上给出的~~没用解释~~，一句话概括：ASM是一个修改，分析Java类文件的框架。先抛开ASM框架的基本定义，先来看看字节码是什么

>字节码（Byte-code）是一种包含执行程序，由一序列 op 代码/数据对组成的二进制文件

又是一段~~废话~~。从这里可以的知，字节码是一种介于翻译语言和底层语言的东西，与底层语言（C/C++）相比较你可以从它这里知道程序的运行方式，而与翻译语言（JavaScript）对比来看，你又无法从字节码中轻易看出什么。但是由于字节码的这个特性，我们得以修改它，操纵它，并且我们还可以反编译它。

***

![ASM图标](/resources/2021030701/asm_picture.jpg)

官网：<https://asm.ow2.io/>

## 添加ASM库依赖

首先，是有关于ASM的Maven依赖
```XML
<dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-all</artifactId>
    <version>6.0_BETA</version>
</dependency>
```

或者

```XML
<dependency>
	<groupId>org.ow2.asm</groupId>
	<artifactId>asm</artifactId>
	<version>9.0</version>
</dependency>
<dependency>
	<groupId>org.ow2.asm</groupId>
	<artifactId>asm-commons</artifactId>
	<version>9.0</version>
</dependency>
<dependency>
	<groupId>org.ow2.asm</groupId>
	<artifactId>asm-tree</artifactId>
	<version>9.0</version>
</dependency>
<dependency>
	<groupId>org.ow2.asm</groupId>
	<artifactId>asm-analysis</artifactId>
	<version>9.0</version>
</dependency>
```

如果使用第一种，那么你最高能依赖的版本是6.0_BETA。而第二种是现在仍在持续更新的，它相当于把原先的asm-all库分开成为不同的部分，但是基本的asm库是必须的。

这个库以后会详细讲述，这里先说几个强有力的工具。

## 帮助学习ASM的工具

### ASMifier: 自动生成ASM代码

ASM库中有不少实用类，为了了解晦涩难懂的ASM代码，可以用ASMifier来进行解析。这是一个可执行类，你可以通过java.exe运行它。

<font color="#ff0000">注意：在9.0已经没有这个类了</font>

运行方法：下载ASM的jar（比如asm-all-6.0_beta.jar）或者从你的.m2文件夹（asm-all）里面找到它，然后运行：

```SH
java -classpath asm-all-6.0_BETA.jar org.objectweb.asm.util.ASMifier DemoDump.class
```

你会得到这样的输出：

![运行ASMifier的效果](/resources/2021030701/asmifier_run.png)

### javap.exe: Java自带的字节码解析工具


javap是你在安装JDK时就有的一个程序文件，是JDK的原生字节码解析工具，关于它的使用因篇幅有限不再细说，可以参考这篇文章 <https://www.cnblogs.com/frinder6/articles/5440173.html>

![运行javap的效果](/resources/2021030701/javap_run.png)

### Eclipse插件: Enhanced Class Decompiler


这个插件是反编译器，可以在你没有库的源代码时反编译出源代码进行调试。当然，这个也可以作为你ASM程序的结果测试方案，通常反编译结果会很贴近于源代码，如果相差很大可以换一种方式反编译。

***

## ASM中经常会用到的名词

### 类(型)的不同名称

* 类的二进制名称/类全名/简单名称

这个三个名称是等价的，也就是我们平常说的类名，例如 java.lang.Thread java.lang.Thread$UncaughtExceptionHandler这样的名称。

* 全限定名

这个名称是用于class文件中的名称，其实就是将二进制名称的所有"."换为"/"，这个名称只有非数组引用类型才有。例如 java/lang/Thread   java/io/IOException。

* 类型描述符

类型描述符是有关于class文件内定义字段等的类型的名称。

### 类型描述符

* 原始类型的描述符一一对应

原始类型的描述符都对应相应的一个字母，具体来说是这样的：

```Java
byte -> B
short -> S
int -> I
long -> J
float -> F
double -> D
char -> C
void -> V
boolean -> Z
```

* 非数组的引用类型为 L+全限定名+;

* 数组引用类型为 [+数组内类型的描述符

例子:

```Java
java.lang.Thread -> Ljava/lang/Thread;
java.lang.Object[] -> [Ljava/lang/Object;
int[][] -> [[I
```

### 方法描述符

了解类名称和类型描述符，下面讲一下方法描述符（其实类型描述符和方法描述符统称描述符）

方法描述符是class文件中保存参数类型列表和返回值类型的方式，在各种方法调用的操作码里面都会涉及到。

规则：

1. 格式为 ( \+ 参数列表 \+ ) \+ 返回值
2. 所有类型名称都为类型描述符
3. 参数列表中不需要逗号分隔

下面是抽象含义下的具体例子（省略了参数名称，只保留了参数类型）：

```Java
void a(int,int,int) -> (III)V
String s(double[],boolean) -> ([DZ)Ljava/lang/String;
int[] i(Object) -> (Ljava/lang/Object;)[I
void t() -> ()V
```

### 操作码(OpCode)

Opcode是用于JVM解释运行Java程序的关键。每一个Opcode都有自己独特的含义与操作，如0x60，助记符iadd，将两个int相加。

有一点要注意：操作码其实就是一个数字，我们平时经常看到的`iadd`，`invokestatic`并不是操作码，而是助记符。

而Java中字节码的名称也与操作码有关，因为每个操作码都是用一个字节，所以叫字节码。

每一个字节用来表示一个指令，理论上可以有 256 个操作码。

对于ASM库来说，所有的Opcode都存储于`org.objectweb.asm.Opcodes`里面，其中还有包括它们在什么方法中作用的注释。

有关于所有操作码的网页：<https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html>

***

这是ASM系列的第一篇，之后会持续更新。

bilibili专栏同步： <https://www.bilibili.com/read/cv6875366>