---
title: Java ASM详解：MethodVisitor与Opcode（三）标签，条件结构，循环结构，栈帧
date: 2021-11-06 11:09:19
category:
	- Java
tags:
	- Java
	- ASM
comments: true
toc: true
---
在之前的文章中，我们已经知道了基础的字节码。但是，这些字节码只能构建起一个简单的结构，不能实现循环条件等高级结构。这篇文章将讨论关于程序流程结构的字节码。

## 标签

标签（Label）是用来划明一部分字节码的标识（通常意义上标签就是一个标记点，但是为了接下来的讲述就用它代表一块字节码）。一个标签下的字节码块，应该从操作栈空开始到操作栈被清空结束——也就是说，一个标签代表的字节码块反编译之后应该是完整的一或多条语句。

在通常情况下，javac编译器会把每条单独语句都分配一个标签，这么做的目的是为了输出行号和局部变量名称。

标签也可以在我们使用Java时自己定义，下面的LABEL就是一个标签：

```Java
LABEL: while(true) {
  for(int i = 0; i < 10; i++)
    if(Math.random() > 0.1)
      break LABEL;
}
```

在字节码中，标签代表的字节码块是从这个标签写入开始到下一个标签写入或该方法的字节码读取完毕的一部分字节码。

在ASM库中，标签用`org.objectweb.asm.Label`进行表示，构造方法如下：

```Java
public Label()
```

写入一个Label，需要用到MethodVisitor的方法，方法如下：

```Java
public void visitLabel(final Label label)
```

正如前面所说，两个标签的写入之间的字节码可以看作这个标签代表的一块字节码块。因此，两个visitLabel之间的语句也可以被看作前一个Label代表的一部分字节码区域。

```Java
MethodVisitor mv = ...
Label l1 = new Label();
mv.visitLabel(l1);
// --- l1代表的字节码块开始
mv....
mv....
// --- l1代表的字节码块结束
Label l2 = new Label();
mv.visitLabel(l2);
```

那么标签有什么用呢？

首先它可以保存代码的行号，这就用到了`MethodVisitor::visitLineNumber`这个方法了。

```Java
public void visitLineNumber(final int line, final Label start)
```

第一个参数代表了这条语句的行号，第二个参数就是这条语句的标签。标签必须先于行号被写入，否则就会抛出`IllegalArgumentException`。

其次，它可以保存局部变量的名称。局部变量有作用域，而作用域可以用两个标签指定。在这两个标签之内且在指定局部变量槽位上的变量就是我们要命名的局部变量。写入局部变量的名称使用`MethodVisitor::visitLocalVariable`。

```Java
public void visitLocalVariable(
      final String name,
      final String descriptor,
      final String signature,
      final Label start,
      final Label end,
      final int index)
```

参数的意义分别是：名称、描述符、泛型签名、开始的标签、结束的标签、局部变量槽位。在javac编译生成的类文件中，局部变量名称的写入都要在最后写入。

最后，标签的最重要意义就是它可以用于跳转字节码上。

## 跳转字节码

用于跳转的字节码都使用了`visitJumpInsn`方法：

```Java
public void visitJumpInsn(final int opcode, final Label label)
```

第一个就是字节码，第二个是跳转的目标。字节码决定了是否进行跳转，标签决定了跳转的目的地。

跳转字节码分为两种——比较跳转和无条件跳转。

### 无条件跳转字节码

无条件跳转，也就是goto字节码：

```Java
输入：无
输出：无
方法：visitJumpInsn
参数：标签
使用范例：
	mv.visitJumpInsn(GOTO, label1);
```

goto字节码是当程序运行到这里时，就直接跳转到对应的标签继续执行，通常都是用在循环内部的。

### 比较跳转字节码

比较跳转是大多数条件结构和循环结构使用的字节码，它有四套字节码，分别对应了int比较、int与0比较、对象比较和对象空检测：

#### if_icmp&lt;cond&gt;

&lt;cond&gt; = eq/ne/lt/ge/gt/le

```Java
输入：两个int数据
输出：无
方法：visitJumpInsn
参数：标签
使用范例：
	mv.visitJumpInsn(IF_ICMPEQ, label1);
```

这六个字节码分别对应了两个int数据进行相等、不相等、小于、大于等于、大于、小于等于的比较测试。如果比较成功，就跳转到指定的标签处运行。如果比较不成功，就沿着当前的流程继续运行。

#### if&lt;cond&gt;

&lt;cond&gt; = eq/ne/lt/ge/gt/le

```Java
输入：一个int数据
输出：无
方法：visitJumpInsn
参数：标签
使用范例：
	mv.visitJumpInsn(IFEQ, label1);
```

这六个字节码分别对应了一个int数据进行等于0、不等于0、小于0、大于等于0、大于0、小于等于0的比较测试。测试结果和跳转方式和上文相同。

#### if_acmp&lt;cond&gt;

&lt;cond&gt; = eq/ne

```Java
输入：两个对象
输出：无
方法：visitJumpInsn
参数：标签
使用范例：
	mv.visitJumpInsn(IF_ACMPNE, label1);
```

这两个字节码分别对应两个对象相等和不相等。这个字节码比较的是对象的引用，而不是内部的值——也就是说，即使两个String对象内部存储字符串一样，也不能保证它们的检测结果为真！（例外是使用了`String::intern`，它会把字符串放进常量池，并返回一个固定的引用）所以判断字符串相等必须使用`equals`方法而不是`==`。

#### ifnull/ifnonnull

```Java
输入：一个对象
输出：无
方法：visitJumpInsn
参数：标签
使用范例：
	mv.visitJumpInsn(IFNULL, label1);
```

这两个字节码分别测试对象是空还是非空。执行流程和之前3个一样。

可以看到，这几个字节码指针对了int和对象引用的情况，而没有考虑long、float、double的情况。于是Java加入了下面几个字节码用于比较它们，获取到值后就可以传递给各个IF字节码判断。

### 比较字节码

#### lcmp

```Java
输入：两个long
输出：一个int
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(LCMP);
```

它用于比较两个long的大小：如果第一个数字比第二个小，返回-1；如果第二个数字比第一个小，返回1；如果相等，返回0。

#### xcmp&lt;op&gt;

x=f/d, &lt;op&gt;=l/g

```Java
输入：两个浮点数
输出：一个int
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(FCMPG);
```

这套字节码和lcmp的逻辑差不多：如果第一个数小于第二个数，返回-1；如果第二个数小于第一个数，返回1；如果相等，返回0。但是，如果其中一方是`NaN`，&lt;op&gt;就决定了它们返回的值：l版本返回-1，而g版本返回1。

接下来，我们将用这22个字节码实现程序的复杂流程结构。

## 条件结构

在Java中，条件结构类似于下面：

```Java
if (condition1) {
  // ...
} else if (condition2) {
  // ...
  // N 个else if
} else if (conditionN) {
  // ...
} else {
  // ...
}
```

在编译期间，这种代码可以被看为：

```Java
if (condition1) {
  // ...
} else 
  if (condition2) {
  // ...
  } else
  	// N 个else if..
    if (conditionN) {
  		// ...
    } else {
  		// ...
  	}
```

也就是说，这种结构就是由一个一个的`if...else`结构组合形成的。一个简单的if...else结构用字节码写入后可以表示为这样的流程：

<image src="/resources/2021110601/if-else.svg" type="image/svg+xml" width=400 height=400/> 

if判断条件通常都使用了返回boolean的表达式（除了特殊字节码指定的比较方式外都需要这样传入），而boolean值的true是1，false是0，使用IFEQ字节码相当于被**反向判断**。相类似的，javac在编译时经常将字节码**操作反转来保证if块先于else块写入**。

返回boolean值传入if中的选择结构类似于这样：

```Java
...返回一个boolean
ifeq label2
label1:
  if块内内容
  goto label3
label2:
  else块内内容
label3:
if块外部代码
```

在跳转之后，我们的操作栈和局部变量表会和跳转之前相等。同时，visitMaxs的参数变成了**所有分支下最大的局部变量表大小和最大的操作栈深度**。

下面举一个例子。要生成这样的Java代码：

```Java
public static int max(int a, int b) {
  if(a > b)
    return a;
  else
    return b;
}
```

我们的字节码应该像下面这样写：

```Java
// 省略ClassWriter和MethodVisitor创建和其他内容
mv.visitVarInsn(ILOAD, 0);
mv.visitVarInsn(ILOAD, 1); // 加载a、b
Label label1 = new Label(); // 创建标签
mv.visitJumpInsn(IF_ICMPLE, label1); // a > b 的反转判断（小于等于），这样能保证if块在前
// if部分内容
mv.visitVarInsn(ILOAD, 0);
mv.visitInsn(IRETURN); // return a 方法结束
mv.visitLabel(label1);
// else部分内容
mv.visitVarInsn(ILOAD, 1);
mv.visitInsn(IRETURN); // return b 方法结束
mv.visitMaxs(2, 2); // 两个int变量，操作栈深度最大2
```

条件结构可以被简化为三元运算符，三元运算符的字节码也类似于if...else。

下面是一个使用三元运算符的例子：

```Java
public static int compute(int val) {
  int endVal = val < 0 ? -val : val;
  return endVal * 2;
}
```

字节码写入：

```Java
// 省略ClassWriter和MethodVisitor创建和其他内容
Label labelElse = new Label(); // else 块的标签
mv.visitJumpInsn(IFGE, labelElse); // 反转判断val
// 三元运算符中前面的表达式
mv.visitVarInsn(ILOAD, 0); 
mv.visitInsn(INEG);
Label labelIfEnd = new Label(); // if...else之后的标签
mv.visitJumpInsn(GOTO, labelIfEnd); // 结束，跳转
mv.visitLabel(labelElse);
// 三元运算符后面的表达式
mv.visitVarInsn(ILOAD, 0);
mv.visitLabel(labelIfEnd);
// 保存到endVal中
mv.visitVarInsn(ISTORE, 1);
// 计算 endVal * 2
mv.visitVarInsn(ILOAD, 1);
mv.visitInsn(ICONST_2);
mv.visitInsn(IMUL);
mv.visitInsn(IRETURN);
mv.visitMaxs(2, 2);
```

## 循环结构

循环结构都比较类似，都是流程返回到之前的代码部分。先从最简单的`while`语句开始：

### while循环

while循环的流程类似与这样：

<image src="/resources/2021110601/while.svg" type="image/svg+xml" width=400 height=400/> 

接下来用一段Java代码写一个例子：

```Java
public static int test() {
	int sum = 0;
	int now = 0;
	while(sum < 1000) {
		sum += ++now;
	}
	return now;
}
```

这是用字节码的方式写入：

```Java
mv.visitInsn(ICONST_0);
mv.visitVarInsn(ISTORE, 0); // 存入sum
mv.visitInsn(ICONST_0);
mv.visitVarInsn(ISTORE, 1); // 存入now
Label labelCondition = new Label();
mv.visitLabel(labelCondition); // 循环条件
mv.visitVarInsn(ILOAD, 0); // 加载sum
mv.visitIntInsn(SIPUSH, 1000);
Label labelEnd = new Label();
mv.visitJumpInsn(IF_ICMPGE, labelEnd); // 反转比较sum < 1000
mv.visitVarInsn(ILOAD, 0);
mv.visitIincInsn(1, 1);
mv.visitVarInsn(ILOAD, 1);
mv.visitInsn(IADD);
mv.visitVarInsn(ISTORE, 0);
mv.visitJumpInsn(GOTO, labelCondition); // 流程跳转回到条件处
mv.visitLabel(labelEnd);
mv.visitVarInsn(ILOAD, 1);
mv.visitInsn(IRETURN);
mv.visitMaxs(2, 2);
```

### do...while循环

类似于while，`do...while`结构也用了和它基本一致的思路，只不过是循环条件写到了后面，并且循环条件**不用反转**：

<image src="/resources/2021110601/do-while.svg" type="image/svg+xml" width=400 height=400/>

do...while循环就不再举例子了，下面来看看for循环。

### for循环

`for`循环有两种：普通的for语句和`for-each`语句。普通的for循环语句在定义时包括了三条语句：一条初始化、一条条件和一条循环结束执行语句。它的流程类似于while多加了一些部分：

<image src="/resources/2021110601/for.svg" type="image/svg+xml" width=400 height=400/>

接下来就是一个例子，使用for循环：

```Java
public static int test() {
	int sum = 0;
	for(int i = 0; i < 100; i++)
		sum += i;
	return sum;
}
```

在字节码里面要这样写：

```Java
mv.visitInsn(ICONST_0);
mv.visitVarInsn(ISTORE, 0); // 存入sum
mv.visitInsn(ICONST_0);
mv.visitVarInsn(ISTORE, 1); // int i = 0，for循环初始化语句
Label labelCondition = new Label();
mv.visitLabel(labelCondition);
mv.visitVarInsn(ILOAD, 1); // i
mv.visitIntInsn(BIPUSH, 100); // 100
Label labelEnd = new Label();
mv.visitJumpInsn(IF_ICMPGE, labelEnd); // i < 100的反转
mv.visitVarInsn(ILOAD, 0); // sum
mv.visitVarInsn(ILOAD, 1); // i
mv.visitInsn(IADD);
mv.visitVarInsn(ISTORE, 0);
mv.visitIincInsn(1, 1); // i++， for循环结束语句
mv.visitJumpInsn(GOTO, labelCondition); // 回到条件
mv.visitLabel(labelEnd);
mv.visitVarInsn(ILOAD, 0);
mv.visitInsn(IRETURN);
mv.visitMaxs(2, 2);
```

另一种for循环，即for-each循环，它的实现和for循环很不一样。

for-each需要一个`Iterable`的对象才能使用，它的原理就是通过`iterator`进行迭代。下面这两种形式是等价的：

```Java
// 设 T extends Iterable<E>
T set = ...;
// for-each写法
for(E element : set) {
	...
}
// 等价写法
Iterator<E> iterator = set.iterator();
while(iterator.hasNext()) {
	E element = iterator.next();
	...
}
```

也就是说，**for-each本质是while循环**。由于没有讲泛型，所以就不细讲此处。

## 栈帧

我相信你已经把上面的例子都跑了一遍（没跑也没事，我默认已经跑了），可是这些东西在你尝试运行它们的时候都会报错。它们报的错无一例外都是`VerifyError`，这是出了什么毛病？这就有关于栈帧了。

Java中执行方法时，JVM会分配给当前线程一个栈帧，栈帧和方法绑定，它的内部就是现在的局部变量表和操作栈数据（这在第三篇文章说过）。栈帧内的局部变量表大小和操作栈大小来自`visitMaxs`。栈帧在方法开始执行时创建，在方法返回时（包括抛出异常）销毁。

但是在类文件中，我们不能保证一个类它的数据是不是异常的——有可能它规定的栈帧局部变量表或者操作栈小于真正运行时的大小。所以Java引入了类的`验证阶段`，检查类内部数据。其中有一项就是检查方法栈帧——检查方法字节码是否正确排序、变量类型是不是一致等。但是这种验证很耗费时间，所以JVM验证器引入了`StackMapTable`进行辅助，这样就能在线性的运行下检查。但是每一行都加入`栈帧映射（stack map frame）`实在是太浪费空间了，所以JVM做了优化，规定每个跳转目标之后都必须有一个映射用于表示栈帧变化。

栈帧映射中并不是一个真的局部变量表和操作栈类型表，它是以一种和前面的映射**比较**的方式保存——比如这个映射要比前面的映射少两个元素等。第一个映射前面并没有别的映射，所以它和**空的操作栈与参数列表**组成的局部变量表的栈帧比较。

（可以看看<https://stackoverflow.com/questions/25109942/what-is-a-stack-map-frame>下面的评论）

所以引发异常的真正原因我们找到了——看来验证器没有检查到方法内部跳转指令后的栈帧映射，导致了验证失败抛出异常。

那么怎么写入栈帧映射呢？

`MethodVisitor`提供了一个方法，叫`visitFrame`。它就是用于写入当前栈帧数据变化的方法。这个方法需要在**每个跳转目标的visitLabel**后面去写，不是用于跳转的标签不需要visitFrame。

visitFrame的方法原型如下：

```Java
public void visitFrame(
      final int type,
      final int numLocal,
      final Object[] local,
      final int numStack,
      final Object[] stack)
```

它有5个参数，指明了这个映射和前面的映射的比较方式和数据。先讲后面的参数，最后再讲第一个参数。

第三个参数是一个代表局部变量变化的一个数组，长度应该为第二个参数。数组内的取值分为这几种：

1. 如果变量是一个没有初始化的对象，那么这个值是指向这个对象`NEW`字节码的标签对象。

2. 如果变量是this并且在调用父类构造函数之前被调用，这个值是`UNINITIALIZED_THIS`。

3. 如果变量类型不是基本类型，值就应该是它的类的全限定名/描述符字符串。

4. 如果是基本类型，那么取值是固定的：int用`INTEGER`代替，float用`FLOAT`代替，long用`LONG`代替，double用`DOUBLE`代替，空用`NULL`代替。long和double即使需要占两个槽位也不需要写两遍，byte、short、char、boolean要用`INTEGER`代替。

5. 如果这个局部变量槽位上暂时是空位（注意不是空对象），用`TOP`代替。

第五个参数类似，是表示操作栈变化的一个数组，长度是第四个参数。

下面是重点——第一个参数的意义。它的不同取值和意义如下：

1. `F_NEW`，只能在Java 6使用（或者ClassWriter被指定扩展栈帧映射），它的写入和之后的版本不一样（其实是类似F_FULL，写入和之前的栈帧信息无关）。这篇文章不会介绍Java 6的栈帧映射写入。

2. `F_SAME`，代表这里的局部变量表和之前的栈帧信息相比没有变化，numLocal和numStack为0，两个数组都为null。（即使不是null也不会写入）

3. `F_SAME1`，代表这里的局部变量表和之前的栈帧信息一样，而操作栈上有一个变量。numLocal是0，numStack是1，local是null，stack是一个数组，内部只有一个元素，代表现在栈上对象的类型。

4. `F_APPEND`，代表现在的局部变量表和之前的栈帧信息一样，但是会多出1-3个新的局部变量。numLocal是新增加的局部变量的数量，local是一个长度为numLocal的数组，存储新增加的局部变量的类型。numStack是0，stack为null。

5. `F_CHOP`，代表现在的局部变量表要比之前的栈帧信息少1-3个局部变量。numLocal就是局部变量缺少的数量，numStack是0，local和stack都是null。

6. `F_FULL`，这代表现在的栈帧和之前的栈帧没有关系，相当于复写了栈帧的信息。numLocal是局部变量数量，local是局部变量类型数组，numStack是操作栈深度，stack是操作栈类型数组。当现在的栈帧比之前的栈帧多/少3个以上的局部变量，或者操作栈上有变量（除非局部变量表不变且栈深度为1可以使用F_SAME1对应），都需要用这个标志重新写入。

在编译时，编译器会尽量减少F_FULL的出现次数，保证类文件不会因为额外栈帧信息变得臃肿。在我们自己生成字节码时，也尽量不要用F_FULL。

接下来，我们来回顾我们报错的代码：

```Java
public static int test() {
	int sum = 0;
	for(int i = 0; i < 100; i++)
		sum += i;
	return sum;
}
```

按照之前的代码，我们要在每个跳转目标上加上栈帧信息：

```Java
// 初始栈帧信息 局部变量表 []
mv.visitInsn(ICONST_0);
mv.visitVarInsn(ISTORE, 0); // 存入sum
mv.visitInsn(ICONST_0);
mv.visitVarInsn(ISTORE, 1); // int i = 0，for循环初始化语句
Label labelCondition = new Label();
mv.visitLabel(labelCondition);
// 这个标签是跳转目标，加入visitFrame
// 这里的栈帧信息 局部变量表 [I I] 操作栈 []
mv.visitFrame(F_APPEND, 2, new Object[] { INTEGER, INTEGER }, 0, null);
mv.visitVarInsn(ILOAD, 1); // i
mv.visitIntInsn(BIPUSH, 100); // 100
Label labelEnd = new Label();
mv.visitJumpInsn(IF_ICMPGE, labelEnd); // i < 100的反转
mv.visitVarInsn(ILOAD, 0); // sum
mv.visitVarInsn(ILOAD, 1); // i
mv.visitInsn(IADD);
mv.visitVarInsn(ISTORE, 0);
mv.visitIincInsn(1, 1); // i++， for循环结束语句
mv.visitJumpInsn(GOTO, labelCondition); // 回到条件
mv.visitLabel(labelEnd);
// 跳转目标，加入栈帧信息
// 这里的栈帧信息 局部变量表 [I] 操作栈 []
mv.visitFrame(F_CHOP, 1, null, 0, null);
mv.visitVarInsn(ILOAD, 0);
mv.visitInsn(IRETURN);
mv.visitMaxs(2, 2);
```

为了方便用户操作，asm自己加了一个计算栈帧信息的标识：`COMPUTE_FRAMES`。在ClassWriter构造函数中使用。

```Java
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
```

使用这个后，所有的visitFrame和visitMaxs都不需要我们自己写。ClassWriter会根据字节码推断栈帧信息等并写入，代价是增加近一倍的运行时间。

## 实战

下面，我们将用字节码写出一个简单的阶乘程序，使用for循环计算阶乘并且用if判断是否溢出。对应的Java代码如下：

```Java
public static long secureFactorial(long number) {
	long result = 1;
	for (int i = 1; i <= number; i++) {
		result *= i;
		if (result < 0) // Overflow
			throw new IllegalArgumentException("Overflow!");
	}
	return result;
}
```

首先计划一下程序标签的位置：

```Java
public static long secureFactorial(long number) {
	long result = 1;
	for (int i = 1; /* labelCondition*/ i <= number; i++) {
		result *= i;
		if (result < 0) // Overflow
			throw new IllegalArgumentException("Overflow!");
		/* labelNoError */
	}
	/* labelEnd */
	return result;
}
```

下面我们用不开启COMPUTE_FRAMES的ClassWriter进行写入：

```Java
// 初始帧 局部变量 [J]
mv.visitInsn(LCONST_1);
mv.visitVarInsn(LSTORE, 2); // 存入result
mv.visitInsn(ICONST_1);
mv.visitVarInsn(ISTORE, 4); // 存入i
Label labelCondition = new Label();
mv.visitLabel(labelCondition);
// 栈帧信息 局部变量 [J J I] 操作栈 []
mv.visitFrame(F_APPEND, 2, new Object[]{ LONG, INTEGER }, 0, null);
mv.visitVarInsn(ILOAD, 4); // i
mv.visitInsn(I2L); // 拉长比较
mv.visitVarInsn(LLOAD, 0); // number
mv.visitInsn(LCMP);
Label labelEnd = new Label();
mv.visitJumpInsn(IFGT, labelEnd); // 反向比较i <= number
mv.visitVarInsn(LLOAD, 2); // result
mv.visitVarInsn(ILOAD, 4); // i
mv.visitInsn(I2L); // 拉长计算
mv.visitInsn(LMUL); // 相乘
mv.visitVarInsn(LSTORE, 2); // 存入result
mv.visitVarInsn(LLOAD, 2); // result
mv.visitInsn(LCONST_0);
mv.visitInsn(LCMP);
Label labelNoError = new Label();
mv.visitJumpInsn(IFGE, labelNoError); // 反向比较 result < 0
// 创建异常对象抛出
mv.visitTypeInsn(NEW, "java/lang/IllegalArgumentException");
mv.visitInsn(DUP);
mv.visitLdcInsn("Overflow!");
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/IllegalArgumentException", "<init>", "(Ljava/lang/String;)V", false);
mv.visitInsn(ATHROW);
mv.visitLabel(labelNoError);
// 栈帧信息 局部变量 [J J I] 操作栈 []
mv.visitFrame(F_SAME, 0, null, 0, null);
mv.visitIincInsn(4, 1); // i++
mv.visitJumpInsn(GOTO, labelCondition); // 回到判断条件
mv.visitLabel(labelEnd);
// 栈帧信息 局部变量 [J J] 操作栈 []
mv.visitFrame(F_CHOP, 1, null, 0, null);
mv.visitVarInsn(LLOAD, 2); // result
mv.visitInsn(LRETURN); // 返回
mv.visitMaxs(4, 5); // 最大操作栈深度4（两个long比较），局部变量5个槽位
```

然后我们对生成的类进行测试：

```Java
Class<?> generated = new Loader().defineClassNow("Test", cw.toByteArray());
System.out.println(generated.getMethod("secureFactorial", long.class).invoke(null, 20));
System.out.println(generated.getMethod("secureFactorial", long.class).invoke(null, 50));
```

得到下面的输出：

```shell
2432902008176640000
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at io.github.nickid2018.asmtest.ASMMain.main(ASMMain.java:60)
Caused by: java.lang.IllegalArgumentException: Overflow!
	at Test.secureFactorial(Unknown Source)
	... 5 more
```

这就代表成功了！

全部源代码：<https://paste.ubuntu.com/p/Gyhn3wHMQ3/>

***

下一期：Java ASM详解：MethodVisitor与Opcode（四）其他流程结构

这篇文章一共讲了22个字节码，加上以前讲过的一共186个。

有错误可以在评论区指出