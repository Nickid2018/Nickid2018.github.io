---
layout: java
title: Java ASM详解：MethodVisitor与Opcode（四）其他流程结构
date: 2021-11-20 18:02:53
toc: true
comments: true
mathjax: true
category:
	- Java
tags:
	- Java
	- ASM
---

上一篇文章中，我们已经了解了基本的流程结构。这篇专栏将详细描述Java中其他的流程结构。

## 异常捕获结构

在平常我们使用流程结构时，除了选择结构和循环结构外，使用最多的大概就是异常捕获结构了。

异常捕获结构的写入都使用了`visitTryCatchBlock`方法（内部的实现是`JSR`和`RET`字节码），它需要早于其他所有字节码写入，也就是在方法写入一开始就要定义。它的定义如下：

```Java
public void visitTryCatchBlock(final Label start, final Label end, final Label handler, final String type)
```

其中，`start`是try块开始的标签；`end`是try结束后的标签（try的范围不包括这个标签）；`handler`是try块内抛出`Throwable`对象后跳转到的标签，即相应的catch块标签；`type`是catch接受的异常类型，要求传入异常类的全限定名（例外是finally块）。

### try-catch

在讲述完整的`try-catch-finally`块之前，我们先来看看普通的`try-catch`块怎么写入。

普通的try-catch块类似这样：

```Java
try {
  // try块
} catch (FirstException e) {
  // 处理FirstException
} catch (SecondException e) {
  // 处理SecondException
}
```

对于一个指定的try块，可能有多个catch块和它对应。每一个catch块都需要用一次visitTryCatchBlock声明。对于catch块对应的跳转标签目标，它的栈帧信息应该和try块前的局部变量相同，但是操作栈上有一个对应的异常对象。下面给出了使用try-catch块的例子：

```Java
public static int parseIntSafely(String s) {
	try {
		/* labelTryStart */
		return Integer.parseInt(s);
	} catch (NumberFormatException e) {
		/* labelCatch */
		return 0;
	}
}
```

使用asm写入如下:

>javac编译时生成的字节码和这里不太一样——它会把已经在操作栈上的Throwable对象先存入局部变量，这是为了输出文件的行号。而这里我们选择直接忽视栈上的Throwable对象。

```Java
// 初始栈帧信息 [java/lang/String]
Label labelTryStart = new Label();
Label labelCatch = new Label();
mv.visitTryCatchBlock(labelTryStart, labelCatch, labelCatch, "java/lang/NumberFormatException"); // try的范围是[labelTryStart, labelCatch)
mv.visitLabel(labelTryStart);
// try块内部
mv.visitVarInsn(ALOAD, 0); // 加载s
mv.visitMethodInsn(INVOKESTATIC, "java/lang/Integer", "parseInt", "(Ljava/lang/String;)I", false); // Integer::parseInt
mv.visitInsn(IRETURN);
// try结束
mv.visitLabel(labelCatch);
// catch块开始
// 栈帧信息 [java/lang/String] [java/lang/NumberFormatException]
mv.visitFrame(F_SAME1, 0, null, 1, new Object[] { "java/lang/NumberFormatException" });
mv.visitInsn(ICONST_0);
mv.visitInsn(IRETURN);
mv.visitMaxs(2, 1); // 最大栈帧为2 （catch块内），局部变量1
```

### multi-catch

除了普通的catch外，还有一种`multi-catch`结构：

```Java
try {
	// try块
} catch (FirstException | SecondException e) {
	// catch块
}
```

multi-catch可以看做几个catch块被共用，这时栈帧信息上的操作栈压入的是multi-catch中所有异常类的**共有超类**。例如一个multi-catch块能捕获`NumberFormatException`和`NullPointerException`，它的字节码写入如下：

```Java
Label labelTryStart = new Label();
Label labelCatch = new Label();
mv.visitTryCatchBlock(labelTryStart, labelCatch, labelCatch, "java/lang/NumberFormatException"); // try的范围是[labelTryStart, labelCatch)
mv.visitTryCatchBlock(labelTryStart, labelCatch, labelCatch, "java/lang/NullPointerException");
// 代码和try块，略
mv.visitLabel(labelCatch);
mv.visitFrame(F_XXXX, X, XXX, 1, new Object[] { "java/lang/RuntimeException" }); // 栈帧其他信息忽略，操作栈上是共有超类RuntimeException
// catch块及之后的代码...
```

### try-finally

说完了try-catch，我们再看看try-finally语句。

```Java
try {
	// try块内容
} finally {
	// finally块内容
}
```

finally其实类似catch，它们都会在操作栈上压入Throwable对象（如果产生了异常），但是它是**无跳转条件（null）**的，无论是否出现异常都会执行一次finally，即**代码流必须经过finally**。如果try块内包含return，也必须先执行finally的内容之后再执行return。如果finally中含有return，则try内的所有return将被忽略，通常IDE会对这种情况给出警告。

在finally执行之后，如果是没有发生异常进入finally，则正常向下运行；如果是因为异常进入了finally，那么在finally执行之后必须抛出异常————这就意味着你必须把finally的字节码**重复两遍**，一次没有异常进入finally，一次发生异常进入finally。

下面是一个例子：

```Java
public static int parseIntAndHello(String s) {
	try {
		/* labelTry */
		return Integer.parseInt(s);
		/* labelReturn */
	} finally {
		/* labelFinally */
		System.out.println("hello");
	}
}
```

asm写入：
>javac编译还是不是这样，但是运行结果是一样的。javac会让一行语句的执行前后操作栈是空，所以在labelReturn前会进行`ISTORE`，在`IRETURN`前`ILOAD`

```Java
Label labelTry = new Label();
Label labelReturn = new Label();
Label labelFinally = new Label();
mv.visitTryCatchBlock(labelTry, labelReturn, labelFinally, null); // try位于[labelTry, labelReturn)，发生任何异常都会跳转finally
mv.visitLabel(labelTry);
// try开始
mv.visitVarInsn(ALOAD, 0); // s
mv.visitMethodInsn(INVOKESTATIC, "java/lang/Integer", "parseInt", "(Ljava/lang/String;)I", false); // Integer::parseInt
// try结束，无异常进入finally
mv.visitLabel(labelReturn);
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("hello");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false); // System.out.println
mv.visitInsn(IRETURN);
// 正常进入finally结束
mv.visitLabel(labelFinally);
// 发生异常进入finally
mv.visitFrame(F_SAME1, 0, null, 1, new Object[] { "java/lang/Throwable" });
// finally接受Throwable对象
mv.visitVarInsn(ASTORE, 1);
// 保存到槽位1用于之后抛出
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("hello");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
// 重复finally内容
mv.visitVarInsn(ALOAD, 1);
// 取出Throwable对象
mv.visitInsn(ATHROW);
// 抛出
mv.visitMaxs(3, 2);
```

### try-catch-finally

接下来，我们把`try-catch`和`try-finally`整合到一起。

finally块的意义是无论发生什么异常都要保证执行，所以catch块的异常也会被finally接受。也就是说，一个完整的try-catch-finally语句需要$1+2catch$次visitTryCatchBlock，并且需要重复finally块字节码$1+catch$次。（其中$catch$是catch块的数量）

下面是个整合的例子：

```Java
public static int parseIntSafelyHello(String s) {
	try {
		/* ltry */
		return Integer.parseInt(s);
		/* lreturn */
	} catch (NumberFormatException e) {
		/* lcatch */
		return -1;
		/* lcatchRet */
	} finally {
		/* lfinally */
		System.out.println("hello");
	}
}
```

asm写入：

```Java
Label ltry = new Label(), lreturn = new Label(), lcatch = new Label(), lcatchRet = new Label(), lfinally = new Label();
mv.visitTryCatchBlock(ltry, lreturn, lcatch, "java/lang/NumberFormatException");
mv.visitTryCatchBlock(ltry, lreturn, lfinally, null);
mv.visitTryCatchBlock(lcatch, lcatchRet, lfinally, null);
mv.visitLabel(ltry);
// try块开始
mv.visitVarInsn(ALOAD, 0); // s
mv.visitMethodInsn(INVOKESTATIC, "java/lang/Integer", "parseInt", "(Ljava/lang/String;)I", false); // Integer::parseInt
// try结束，try无异常进入finally
mv.visitLabel(lreturn);
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("hello");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false); // System.out.println
mv.visitInsn(IRETURN);
// try无异常finally结束
mv.visitLabel(lcatch);
// catch块开始
mv.visitFrame(F_SAME1, 0, null, 1, new Object[] { "java/lang/NumberFormatException" });
// 栈帧信息：压入异常对象
mv.visitInsn(POP);
// 我们不需要这个对象,直接弹栈（javac会进行ASTORE，无论是否使用，因为要记录名称）
mv.visitLabel(lcatchRet);
// catch无异常进入finally
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("hello");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false); 
// 重复一遍finally...
mv.visitInsn(ICONST_M1);
mv.visitInsn(IRETURN); // 返回-1
mv.visitLabel(lfinally);
// 发生异常跳转finally
mv.visitFrame(F_SAME1, 0, null, 1, new Object[] { "java/lang/Throwable" });
mv.visitVarInsn(ASTORE, 1); // 保存到槽位1
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("hello");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false); 
// 再重复一遍finally...
mv.visitVarInsn(ALOAD, 1);
mv.visitInsn(ATHROW);
// 抛出
mv.visitMaxs(3, 2);
```

### try-with-resources

除了try-catch、multi-catch、try-finally、try-catch-finally结构外，还有一种结构：`try-with-resources`。这种结构要求一个`AutoClosable`的对象在try后的语句中初始化：

```Java
try (AutoClosable res1 = getObject1(), res2 = getObject2(), ...) {
	// try内的语句
} catch (Exception e) {
	// catch块
} finally {
	// finally...
}
```

它可以转化为普通的try-catch-finally块，类似于这样：

>javac编译之后内部不是这样，这里是将执行流程强制转换为可读Java源码

```Java
AutoClosable res = null;
try {
	res = getObject1();
	// try块内容...
} catch (Exception e) {
	// catch块内容...
	// 这里res也会被close，但是因为Java代码没法表现就没有写
} catch (Throwable t) {
	try {
		res.close(); // 这里其实res已经被创建了，但是因为Java代码没法表现就放在这里了
	} catch (Throwable t2) {
		t.addSupressed(t2);
	}
	throw t;
} finally {
	// finally...
}
```

使用try-with-resources结构写入字节码的时候，只要记住每一个出口都会进行一次带try-catch的`close`就可以。由于这种结构很复杂且代码量巨大，就不举例子了。

## switch多分支结构

在一些情况下，`if...else if...else`结构非常的长，这时我们可以用`switch`替代。

### 可直接表示为int的switch多分支语句

最简单的switch是键为整形数字常量（可以用int表示的）的，类似于这样：

```Java
int i = ...;
switch (i) {
	case 0:
	case 1:
		...
	...
	default:
		...
}
```

在写入switch中，我们有两个方法可以选择：

```Java
public void visitTableSwitchInsn(final int min, final int max, final Label dflt, final Label... labels)
public void visitLookupSwitchInsn(final Label dflt, final int[] keys, final Label[] labels)
```

它们的相同之处是：它们都需要操作栈顶上有一个int类型的值。它们的不同之处在于它们对于键值的存储方式和使用的字节码：

* `visitTableSwitchInsn`写入的键值是一个连续的数组——一个$[min,max]$的一个整形数字数组。如果switch中没有中间的某些键值，那么这些键值会和`dflt`一致，即default的标签（如果没有default块，则dflt应该指向switch结束后的第一条语句）。它使用`TABLESWITCH`字节码。

* `visitLookupSwitchInsn`要求传入一个switch键值的数组，数组内的数字要从小到大排序。`labels`数组的长度要与`keys`一致。`dflt`也是指向default或者switch结束后的第一条语句的标签。它使用`LOOKUPSWITCH`字节码。

回到最简单的switch上来。我们需要按照键值的特性选择我们的写入方式：

* 如果switch内的键值差异小，并且键值组成一个连续整数数组的空缺不超过6个，则使用visitTableSwitchInsn

* 如果switch内的键值差异大，则使用visitLookupSwitchInsn

先看个简单的小例子：

```Java
public static void test(int i) {
	switch (i) {
		case 0:
		case 1:
        /* label1 */
			System.out.println("1");
		case 2:
        /* label2 */
			System.out.println("2");
			break;
		case 3:
        /* label3 */
			System.out.println("3");
	}
	/* labelEnd */
}
```

可以看到，键值$\{0,1,2,3\}$组成了一个连续的整数数组，所以这里我们应该使用visitTableSwitchInsn。

```Java
mv.visitVarInsn(ILOAD, 0); // i
Label label1 = new Label(), label2 = new Label(), label3 = new Label(), labelEnd = new Label();
mv.visitTableSwitchInsn(0, 3, labelEnd, label1, label1, label2, label3);
// 解释：没有default块所以指向了switch结束的下一条语句，0和1用了一个跳转位置
mv.visitLabel(label1);
// 0和1处理
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("1");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitLabel(label2);
// 2的处理，注意这里0和1处理之后也会经过这个地方
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("2");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitJumpInsn(GOTO, labelEnd); // break的作用，跳出switch
mv.visitLabel(label3);
// 3的处理，这里不会被0、1、2访问了
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("3");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
// switch结束
mv.visitLabel(labelEnd);
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitInsn(RETURN);
mv.visitMaxs(2, 1);
```

这是对于switch最简单的一种清况之一。因为`byte`、`short`、`char`在JVM内解释为int，所以这些步骤基本相同。

### 使用枚举的switch多分支语句

switch语句还可以用于枚举类型，下面我们定义了一个枚举，并使用了它：

```Java
enum TestEnum {
	FIRST, SECOND, THIRD, FOURTH
}

public static void test(TestEnum i) {
	switch (i) {
		case FIRST:
		case SECOND:
			System.out.println("1");
		case THIRD:
			System.out.println("2");
			break;
		case FOURTH:
			System.out.println("3");
		default:
			System.out.println("4");
	}
}
```

枚举类型不能直接作为两种switch字节码的参数，它必须先变为一个int才能传入字节码。为此，javac在编译的时候会自动创建一个内部类，用于保存这个类里面出现的所有使用枚举对象switch的一个映射表。对于我们定义的TestEnum，它对应的映射类应该像这样（假设我们方法定义的类是`Test`）：

```Java
// 注意：这个类和字段都要有ACC_SYNTHETIC访问标志
class Test$1 {

	static final int[] $SwitchMap$TestEnum;
	
	static {
		$SwitchMap$TestEnum = new int[TestEnum.values().length];
		try {
			$SwitchMap$TestEnum[TestEnum.FIRST.ordinal()] = 1;
		} catch (NoSuchFieldError e) {
		}
		try {
			$SwitchMap$TestEnum[TestEnum.SECOND.ordinal()] = 2;
		} catch (NoSuchFieldError e) {
		}
		try {
			$SwitchMap$TestEnum[TestEnum.THIRD.ordinal()] = 3;
		} catch (NoSuchFieldError e) {
		}
		try {
			$SwitchMap$TestEnum[TestEnum.FOURTH.ordinal()] = 4;
		} catch (NoSuchFieldError e) {
		}
	}
}
```

这个内部类中含有所有在这个类中出现的枚举对象，每个枚举类都会创建一个字段，命名为`$SwitchMap$+.替换成$的类型名`，它们的长度是对应枚举类枚举字段的数量，按照ordinal大小排序将`1-n`写入数组（n是本类使用了多少个这个类的枚举字段）。

接下来，switch的传入方式也发生了变化：

```Java
TestEnum i ...
switch (Test$1.$SwitchMap$TestEnum[i.ordinal()]) {
	...
}
```

那么之前给出的例子我们可以用asm写入为：

```Java
mv.visitFieldInsn(GETSTATIC, "Test$1", "$SwitchMap$TestEnum", "[I"); // 获取常量字段
mv.visitVarInsn(ALOAD, 0); // i
mv.visitMethodInsn(INVOKEVIRTUAL, "TestEnum", "ordinal", "()I", false); // Enum::ordinal
mv.visitInsn(IALOAD); // 取出常量
Label label1 = new Label(), label2 = new Label(), label3 = new Label(), labelDefault = new Label(), labelEnd = new Label();
mv.visitTableSwitchInsn(1, 4, labelDefault, label1, label1, label2, label3);
// default标签-labelDefault, 1/2用了同一个跳转目标
mv.visitLabel(label1);
// 1/2 FIRST/SECOND
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("1");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitLabel(label2);
// 3 THIRD
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("2");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitJumpInsn(GOTO, labelEnd); // break
mv.visitLabel(label3);
// 4 FOURTH
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("3");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitLabel(labelDefault);
// default
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("4");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitLabel(labelEnd);
// switch结束
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitInsn(RETURN);
mv.visitMaxs(2, 2);
```

如果是我们自己写入asm，推荐不要用这种方式写入——毕竟太麻烦了。最好的方案是使用`Enum::ordinal`获取序号对序号进行switch，而不是存一个新的表。

### 使用字符串的switch多分支语句

除了枚举和基本int之外，switch还允许字符串传入。下面就是一个例子：

```Java
public static void test(String s) {
	switch (s) {
		case "HELLO":
			System.out.println("hello");
			break;
		case "BYEBYE":
			System.out.println("byebye");
			break;
		default:
			System.out.println("ss");
	}
}
```

很明显，String不能直接转换成为int。在String中，`hashCode`这个方法可以让我们将字符串映射到int上，这样就能把它作为键值。但是还有一个问题需要解决：String和int不能一一对应——不同的字符串可能有相同的hashCode，例如`ddnqavbj`和`166lr735ka3q6`的哈希码值都为`0`。因此，javac在编译时将这个switch块**拆开为两个，并使用一个临时量保存字符串的映射**。这样，上面的例子就变成了下面这样：

```Java
public static void test(String s) {
	{
		String tempStr = s;
		int temp = -1;
		switch (tempStr.hashCode()) {
			case 68624562: // "HELLO"的哈希码
				/* hashHello */
				if (tempStr.equals("HELLO"))
					temp = 0;
				break;
			case 1973839168: // "BYEBYE"的哈希码值
				/* hashBye */
				if (tempStr.equals("BYEBYE"))
					temp = 1;
				break;
		}
		/* switch2 */
		switch (temp) {
			case 0:
				/* case0 */
				System.out.println("hello");
				break;
			case 1:
				/* case1 */
				System.out.println("byebye");
				break;
			default:
				/* caseDefault */
				System.out.println("ss");
		}
	}
	/* end */
}
```

按照上面的Java代码，我们能用asm将它写入：

```Java
// 初始帧 [java/lang/String]
mv.visitVarInsn(ALOAD, 0); // s
mv.visitVarInsn(ASTORE, 1); // 转存到tempStr
mv.visitInsn(ICONST_M1); // -1
mv.visitVarInsn(ISTORE, 2); // 存入temp
mv.visitVarInsn(ALOAD, 1); // tempStr
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/String", "hashCode", "()I", false); // tempStr.hashCode()
Label hashHello = new Label(), hashBye = new Label(), switch2 = new Label();
mv.visitLookupSwitchInsn(switch2, new int[]{ 68624562, 1973839168 }, new Label[]{ hashHello, hashBye });
mv.visitLabel(hashHello);
// HELLO的哈希码
// 栈帧 [java/lang/String java/lang/String I]
mv.visitFrame(F_APPEND, 2, new Object[]{"java/lang/String", Opcodes.INTEGER}, 0, null);
mv.visitVarInsn(ALOAD, 1); // tempStr
mv.visitLdcInsn("HELLO");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/String", "equals", "(Ljava/lang/Object;)Z", false); // 判断
mv.visitJumpInsn(IFEQ, switch2); // 反转判断，失败直接跳转出去
mv.visitInsn(ICONST_0); // 0
mv.visitVarInsn(ISTORE, 2); // 赋值给temp
mv.visitJumpInsn(GOTO, switch2); // 也跳转出去
mv.visitLabel(hashBye);
// BYEBYE的哈希码
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitVarInsn(ALOAD, 1); // tempStr
mv.visitLdcInsn("BYEBYE");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/String", "equals", "(Ljava/lang/Object;)Z", false); // 判断
mv.visitJumpInsn(IFEQ, switch2); // 反转判断，失败直接跳转
mv.visitInsn(ICONST_1); // 1
mv.visitVarInsn(ISTORE, 2); // 赋值给temp
mv.visitLabel(switch2);
// 第一个switch结束
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitVarInsn(ILOAD, 2); // 加载temp
Label case0 = new Label(), case1 = new Label(), caseDefault = new Label(), end = new Label();
mv.visitLookupSwitchInsn(caseDefault, new int[]{ 0, 1 }, new Label[]{ case0, case1 });
// 这里可以用tableswitch，但是javac是这样编译出来的
mv.visitLabel(case0);
// HELLO
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("hello");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitJumpInsn(GOTO, end); // 跳转结束
mv.visitLabel(case1);
// BYEBYE
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("byebye");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitJumpInsn(GOTO, end); // 跳转结束
mv.visitLabel(caseDefault);
// default
mv.visitFrame(F_SAME, 0, null, 0, null); // 栈帧无变化
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("ss");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitLabel(end);
// 结束
// 栈帧信息 [java/lang/String]
mv.visitFrame(F_CHOP, 2, null, 0, null);
mv.visitInsn(RETURN);
mv.visitMaxs(2, 3);
```

### 增强型switch

最后来看看Java 14新加的增强型switch。

首先，增强型switch可以返回一个值赋给变量或者进行操作：

```Java
System.out.println(switch (s) {
	case "HELLO" -> 3;
	case "BYEBYE" -> 4;
	default -> 5;
});
```

这种操作的本质还是和上面的一样，下面是展开增强型switch但是不展开String转换的结果：

```Java
int temp;
switch (s) {
	case "HELLO":
		temp = 3;
		break;
	case "BYEBYE":
		temp = 4;
		break;
	default:
		temp = 5;
}
System.out.println(temp);
```

另一种增强型switch使用了`yield`关键字：

```Java
System.out.println(switch (s) {
	case "HELLO":
		System.out.println("case0");
		yield 3;
	case "BYEBYE":
		System.out.println("case1");
		yield 4;
	default:
		System.out.println("default");
		yield 5;
});
```

它的原理也和上面差不多：

```Java
int temp;
switch (s) {
	case "HELLO":
		System.out.println("case0");
		temp = 3;
		break;
	case "BYEBYE":
		System.out.println("case1");
		temp = 4;
		break;
	default:
		System.out.println("default");
		temp = 5;
}
System.out.println(temp);
```

***

这篇文章到这里就结束了（最后不写例子主要是因为这两种结构需要的代码量太大了）。

这回一共讲了4个字节码，加上以前的一共190个。

有错误可以在评论区指出~

下一期 Java ASM详解：MethodVisitor与Opcode（五）invokedynamic、方法引用、BSM