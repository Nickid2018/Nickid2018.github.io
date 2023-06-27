---
title: Java ASM详解：MethodVisitor与Opcode（一）基本操作与运算
layout: default
date: 2021/4/7
updated: 2021/8/29
category:
	- Java
tags:
 - Java
 - ASM
comments: true
toc: true
---

前文我们说到了很多Visitor，它们用于给类中定义类型，添加字段，附上注释。但是对于一个语言来说，最重要的那一部分我们还没有说到——那就是：方法。

## 方法定义

在ClassVisitor中，我们看到了有一个方法名为visitMethod，参数是(int，String，String，String，String[])，按照参数列表的顺序，它们分别指**访问标志，方法名，方法描述符，泛型签名和抛出异常列表**，返回一个MethodVisitor。（关于方法描述符，请看此系列的第一篇；关于访问标志，请看第二篇）

对于方法名，有下面的规则：

<font color=blue>1.方法名不能是关键字或保留字（goto）

2.方法名不能以数字开头

3.可以为&lt;init&gt;和&lt;clinit&gt;</font>

其中，`<init>`是构造函数，一个类可以有不止一个构造函数。而`<clinit>`每个类最多有一个，并且方法描述符必须为`()V`，它在类初始化阶段被JVM调用。（包括调用这个类的成员和Class.forName，但不包括Class.forName的initialize参数为false时的调用）

若方法名不正确，在尝试加载这个类的时候会抛出`java.lang.ClassFormatError: Illegal method name`。

抛出异常列表中，所有的类名称都为**全限定名**。

## 操作栈（Operand Stack）

操作栈是一个方法被调用时JVM分配出来的一个栈空间，它用于存储方法内加载的数据和进行字节码指令操作。当JVM接收到一个字节码指令（例如iadd），就会取出栈顶的几项元素（对于iadd来说，就是栈顶的两项），在进行操作之后，将计算或获得的数据放回栈顶（比如iadd计算栈顶两个int的加和之后会放回加和数字）。

对于普通的对象，只会占用一个栈元素。但对于long或double这种对象，会占用两个栈元素。这有关于之后要介绍的visitMaxs。

如果一个字节码需要超过了现在操作栈内的元素数量的元素，那么在调用生成的方法时会抛出`java.lang.VerifyError: Unable to pop operand off an empty stack`。

如果一个字节码需要的类型与现在操作栈中元素类型不同，那么在调用生成的方法时抛出`java.lang.VerifyError: Register <slot> contains wrong type`或`java.lang.VerifyError: Bad type on operand stack`。

在之后的讲解中，我们会大量的使用这个名词，在接下来的编写中，操作栈的变化将会像下面这样写：

```
输入：XXX YYY
输出：ZZZ
```

## 局部变量表（Local Variable Table）

局部变量表在方法调用中分配的另一个空间，用于存储现在方法内所有的局部变量，表中的数据可以被编号为0-n，叫Slot。普通的元素只会占用一个Slot，但long和double这种数据会占用两个。关于这个的详细使用，请看下面的xload和xstore的字节码介绍。

当这个方法为静态方法时，局部变量表会将参数列表中的变量按顺序放入局部变量表中。

当这个方法不是静态方法，局部变量表的0位是this，之后才会将参数列表变量依次放入表中。

如果局部变量表大小超过了256，那么字节码将会发生变化，xload、xstore等都会受到影响（需要以wide字节码辅助才能进行正常的局部变量读取写入）。但是ASM9中不提供wide字节码，因为MethodWriter中有ASM库自己的处理，所以在用户层编写ASM是无影响的。

## MethodVisitor的方法

在说完操作栈的概念之后，我们来看看MethodVisitor中都定义了哪些有关于字节码和执行的方法。

下面这些方法第一个参数都为字节码。

`visitInsn(int)`：访问一个零参数要求的字节码指令，如ACONST_NULL

`visitIntInsn(int, int)`：访问一个需要零操作栈要求但需要有一个int参数的字节码指令，如BIPUSH

`visitVarInsn(int, int)`：访问一个有关于局部变量的字节码指令，如ALOAD

`visitTypeInsn(int, String)`：访问一个有关于类型的字节码指令，如CHECKCAST

`visitFieldInsn(int, String, String, String)`：访问一个有关于字段的字节码，如PUTFIELD

`visitMethodInsn(int, String, String, String, boolean)`：访问一个有关于方法调用的字节码，如INVOKESPECIAL

`visitJumpInsn(int, Label)`：访问跳转字节码，如IFEQ

之后，是一些被包装好的字节码访问方法，这些方法都基于最基本的字节码指令，但是不需要我们自己用上面提到的那些方法直接调用字节码。

`visitInvokeDynamicInsn(String, String, Handle, Object...)`：基于INVOKEDYNAMIC，动态方法调用，会在lambda表达式和方法引用里面说到

`visitLdcInsn(Object)`：基于LDC、LDC\_W和LDC2\_W，将一个常量加载到操作栈用（详细见下文）

`visitIincInsn(int, int)`：基于IINC、IINC_W，自增/减表达式

`visitTableSwitchInsn(int, int, Label, Label...)`：基于TABLESWITCH，用于进行table-switch操作

`visitLookupSwitchInsn(Label, int[], Label[])`：基于LOOKUPSWITCH，用于进行lookup-switch操作

`visitMultiANewArrayInsn(String, int)`：基于MULTIANEWARRAY，用于创建多重维度数组，如int\[\]\[\]

在下文说到它们时，会以下面的方式表达：

```
方法：visitXXXInsn
参数：XXX YYY ZZZ
```

到这里，所有有关于字节码指令的方法就结束了。块级结构的方法会在下一篇说。

最后，说一下每个方法都要在最后调用的方法：visitMaxs(int, int)。它第一个参数是操作栈的最大大小，第二个是局部变量的个数。如果你调用这个方法时局部变量数量写小了，就会在生成方法调用时抛出`java.lang.ClassFormatError: Arguments can't fit into locals`，如果操作栈大小写小了，在生成方法调用时会抛出`java.lang.VerifyError: Stack size too large`

那么下面，我们将逐系列逐条讲解所有的字节码。这篇专栏先讲基本的操作栈加载存储等操作、常量获取和运算操作。

<font color=blue>注意：接下来的x可以为a（针对对象）、i（针对int）、l（针对long）、f（针对float）、d（针对double）、b（针对byte）、c（针对char）、s（针对short），它代表了操作对象的类型。有些时候没有针对于byte和short的专用字节码，这是因为在JVM中，byte和short在被计算时会被强制拉长为int，所以它们使用的和int一样。char和int能互相转换。boolean类似，它们也需要使用int的字节码，而且boolean值的false就是int值0，而true就是int值1。</font>

## 字节码介绍

### 啥事都不干的字节码：nop

```Java
输入：无
输出：无
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(NOP);
```
		
这个字节码啥都不干，在实际开发中可以当做代码插入点使用。

### 加载字节码：<i>x</i>load与<i>x</i>load_<i>n</i>

x=a/i/l/f/d

```Java
输入：无
输出：某一对象或基本数据类型数据
方法：visitVarInsn
参数：加载对象的位置
使用范例：
	mv.visitVarInsn(ALOAD, 5);
	mv.visitVarInsn(FLOAD, 3); // 在javap反汇编中，此处变为fload_3
```

如果在调用此字节码时对应位置没有初始化变量（原先为参数或已经用xstore进行值的放入被视为该位置被初始化），在生成方法调用时会抛出`java.lang.VerifyError: Accessing value from uninitialized register <slot>`。

如果要进行加载的对象位置小于等于3，可以用对应的xload\_n版本代替（注意，ASM9的Opcodes中已经不存在xload\_n版本的字节码常量，但是在javap反汇编时可以看到此条），例如aload_2。

### 存储字节码：<i>x</i>store与<i>x</i>store_<i>n</i>

x=a/i/l/f/d;n=0,1,2,3

```Java
输入：某一对象或基本类型数据
输出：无
方法：visitVarInsn
参数：存储对象的位置
使用范例：
	mv.visitVarInsn(ASTORE, 4);
	mv.visitVarInsn(ISTORE, 1); // 在javap反汇编中，此处变为istore_1
```
		
存储对象的位置规则与加载相同。与加载规则不同的是，xstore可以指定到一个未初始化的位置，并将这个位置初始化。有意思的一点是，你可以不遵循初始化位置的连续性，也就是说，假如2、3位置都未初始化，你可以通过xstore将对象放入3中并初始化它，这时位置2变为了未定义的状态，它在被xload加载时都会抛出`java.lang.VerifyError: Register <slot> contains wrong type`，即使你用的加载指令与放入指令类型相同。这时你只能通过另一次xstore将对象放入位置2，才能使这个位置类型固定。

和xload一样，xstore也有xstore_n版本，但ASM9已经不支持直接写入它们了。

### 返回字节码：<i>(x)</i>return

x=a/i/l/f/d

```Java
输入：某一对象或基本类型数据
输出：清空操作栈并返回
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(ARETURN);
	mv.visitInsn(RETURN); // 无返回，用于void方法
```
		
返回字节码是每个方法必有的，包括void无返回值方法。如果一个方法没有写任何的返回字节码指令，在调用这个生成的方法时就会抛出`java.lang.VerifyError: Falling off the end of the code`。

返回字节码无视操作栈内剩余的所有值，只会将栈顶元素返回，并清除操作栈。

在这个方法为同步方法的前提下，所在线程不是已经锁定的监视器对象所有者时，这条指令会抛出`IllegalMonitorStateException`。这种情况在普通状况下根本无法发生，只有当这个同步方法上在其同步对象上使用了monitorexit却没有使用monitorenter时可能发生。

### 复制栈顶字节码：dup家族

```Java
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(DUP);

每种字节码的解析：
1. DUP
输入：...v1
输出：...v1 v1
2. DUP_X1
输入：...v2 v1
输出：...v1 v2 v1
3. DUP_X2
输入：...v3 v2 v1
输出：...v1 v3 v2 v1
4. DUP2
输入：...v2 v1
输出：...v2 v1 v2 v1
5. DUP2_X1
输入：...v3 v2 v1
输出：...v2 v1 v3 v2 v1
6. DUP2_X2
输入：...v4 v3 v2 v1
输出：...v2 v1 v4 v3 v2 v1
```
	
这个字节码是用于复制栈顶元素并插入到栈中的字节码，可以节省xload和xstore的使用量。在这里，...指栈顶下的其他元素。

DUP家族的名称规律是：DUP后紧接着的数字代表了复制数量，Xn代表插入到栈顶下第几层。

### 弹出栈顶字节码：pop，pop2

```Java
输入：一（pop）或两（pop2）个元素
输出：弹出栈顶一（pop）或两（pop2）个元素
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(POP);
	mv.visitInsn(POP2);
```
		
这个字节码也是用于操作操作栈的。它的使用情况举一个例子：调用了一个有返回值的方法但返回值我们不需要，就可以采用POP。

### 交换元素字节码：swap

```Java
输入：两个元素
输出：交换栈顶两个元素
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(SWAP);
```
		
这个字节码可以交换栈顶的两个操作数。

### 常量池常量读取字节码：ldc（ldc_w, ldc2_w）

```Java
输入：无
输出：从常量池读取出的数据
方法：visitLdcInsn
参数：常量值（见下文） [在JVM中，此处是常量池中对应常量的序号，长度分为三种，由三种LDC指令决定]
使用范例：
	mv.visitLdcInsn("helloworld");
	mv.visitLdcInsn(20.0f); // 注意，此处自动装箱成为Float
	mv.visitLdcInsn(Type.getType("I")); // 类型
```
	
常量池（Constant Pool）中，含有以下几种数据：整数Integer、浮点数Float、字符串字面值常量String、类的引用Type、句柄Handle或动态常量值ConstantDynamic，所以LDC值可能有这些。

在JVM中，如果常量值是Integer或Float，就会直接将它们放到操作栈顶；如果为String，将String类的引用放到操作栈顶；若为Type，将对应的类型初始化，并将其Class实例引用放到操作栈顶；对于Handle，将java.lang.invoke.MethodHandle/MethodType的引用至于操作栈顶。

在解析类型的引用期间（Type），这条指令可能会抛出有关于类加载的异常；同样的，解析有关于句柄（Handle）的时候也有可能抛出和句柄有关的异常。

### 空值常量字节码：aconst_null

```Java
输入：无
输出：常量值null
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(ACONST_NULL);
```
		
当程序中使用了null，就可以用这个字节码。
		
### 普通数字常量字节码：<i>x</i>const_<i>n</i>

x=i/l/f/d;对于iconst，n=m1,0,1,2,3,4,5;对于lconst、dconst，n=0,1;对于fconst，n=0,1,2

```Java
输入：无
输出：数字常量值，类型与字节码有关
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(ICONST_M1); // -1
	mv.visitInsn(FCONST_0); // +0.0f
	mv.visitInsn(LCONST_1); // 1L
```

当数字较小时，获得数字常量可以不使用LDC，可以直接用这些字节码代替（节省常量池空间）。

### 整数常量字节码：bipush和sipush

```Java
输入：无
输出：数字常量值
方法：visitIntInsn
参数：某一具体整数
	对于bipush，数字属于byte范围（-128~127）
	对于sipush，数字属于short范围（-32768~32767）
使用范例：
	mv.visitIntInsn(BIPUSH, 27); // 27
	mv.visitIntInsn(SIPUSH, -2700); // -2700
```

当一个数字没有超过这两个字节码规定的范围，我们都可以使用这两个字节码获取整数常量。在编译中，属于这个范围的数字都是用它们进行获取整数（除非是-1\~5），而更大/小的整数都是用LDC。

说完了基本的加载存储常量指令，下面来看看程序的最基本功能：计算。

### 取反运算字节码：<i>x</i>neg

x=i/l/f/d

```Java
输入：数字
输出：数字的相反数
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(INEG);
	mv.visitInsn(DNEG);
```
		
这个字节码用于计算取反（-x）。注意：如果计算时数字溢出、下溢或精度丢失，这个字节码也不会反馈任何警告。

对于整数（int和long），计算规则就是(\~x)+1，当它们处于MIN\_VALUE时，取反结果仍为MIN\_VALUE。

对于浮点数（double和float），这个字节码运算为：

1. 取反与从零减去不等价，若x为+0.0，0.0-x结果为+0.0，而-x为-0.0

2. 若数字为NaN（Not A Number，float的0x7fc00000或double的0x7ff8000000000000L），结果也为NaN

3. 若数字为无穷大（float正0x7f800000负0xff800000，double正0x7ff0000000000000L负0xfff0000000000000L），结果为相反符号的无穷大

4. 若数字为0，结果为相反符号的0

### 加法运算字节码：<i>x</i>add

x=i/l/f/d

```Java
输入：加数1 加数2
输出：数字的和
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(IADD);
	mv.visitInsn(DADD);
```
		
这个字节码用于计算加法（a+b）。注意：如果计算时数字溢出、下溢或精度丢失，这个字节码也不会反馈任何警告。

对于浮点数（double和float），这个字节码运算为：

1. 如果两个数都为NaN，结果是NaN。

2. 如果两个数为相反符号的无穷大，和为NaN

3. 同一符号的无穷大结果是该符号的无穷大

4. 有限值与无穷大的和还是无穷大

5. 相反符号的两个0（+0.0和-0.0）结果为+0.0

6. 相同符号的两个0和为该符号的0

7. 0与非零值的和为非零值

8. 符号相反，绝对值相等的有限值和为+0.0

9. 若不属于上面的情况，结果将以IEEE 754舍入到最近可表示的浮点值。如果结果太大无法表示为浮点数（超过最大表示范围“溢出”，也就是绝对值超过float的3.4028235e+38f或double的1.7976931348623157e+308），结果为对应符号的无穷大；如果结果太小无法表示为浮点数（超过最小表示范围“下溢”，也就是绝对值小于float的1.4e-45f或double的4.9e-324），结果是对应符号的0。

### 减法运算字节码：<i>x</i>sub

x=i/l/f/d

```Java
输入：被减数 减数
输出：两数字之差
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(ISUB);
	mv.visitInsn(DSUB);
```
		
这个字节码用于计算减法（a-b），等价于a+(-b)。注意：如果计算时数字溢出、下溢或精度丢失，这个字节码也不会反馈任何警告。

浮点数运算法则请同时参照xadd与xneg。

### 乘法运算字节码：<i>x</i>mul

x=i/l/f/d

```Java
输入：乘数1 乘数2
输出：两数字之积
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(IMUL);
	mv.visitInsn(DMUL);
```
		
这个字节码用于计算乘法（a\*b）。注意：如果计算时数字溢出、下溢或精度丢失，这个字节码也不会反馈任何警告。

对于浮点数（double和float），这个字节码运算为：

1. 两个数字中有一个是NaN，结果为NaN

2. 无穷大乘以一个0，结果为NaN

3. 无穷大与有限值相乘，结果为无穷大，符号取决于两个数字的符号是否相同，相同为正，相反为负

4. 其余情况为IEEE 754规定，在xadd那里有完整说明

### 除法运算字节码：<i>x</i>div

x=i/l/f/d

```Java
输入：被除数 除数
输出：两数字之商
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(IDIV);
	mv.visitInsn(DDIV);
```
		
这个字节码用于计算除法（a/b）。注意：如果计算时数字溢出、下溢或精度丢失，这个字节码也不会反馈任何警告。

对于整数（int和long），这个字节码只会保留商的整数部分。如果除数为0，这个字节码会抛出`java.lang.ArithmeticException: / by zero`

对于浮点数（double和float），这个字节码运算为：

1. 两个数字中有一个是NaN，结果为NaN

2. 无穷大除以无穷大，结果为NaN

3. 无穷大除以有限值，结果为无穷大，符号取决于两个数字的符号（规则见xmul）

4. 有限值除以无穷大，结果为0，符号同上

5. 0除以0为NaN

6. 0除以有限值为0，符号同上

7. 有限值除以0为无穷大，符号同上

8. 其余情况为IEEE 754规定，在xadd那里有完整说明

### 取余运算字节码：<i>x</i>rem

x=i/l/f/d

```Java
输入：被除数 除数
输出：余数
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(IREM);
	mv.visitInsn(DREM);
```
		
这个字节码用于计算取余操作（a%b）。注意：如果计算时数字溢出、下溢或精度丢失，这个字节码也不会反馈任何警告。

对于浮点数（double和float），这个字节码运算为：

1. 两个数字中有一个是NaN，结果为NaN

2. 符号取决于被除数

3. 被除数为无穷大或除数为0，结果为NaN

4. 被除数为有限值而除数为无穷大，结果为被除数

5. 被除数为0，结果为0

6. 其余情况为IEEE 754规定，在xadd那里有完整说明

### 自增字节码：iinc（iinc_w）

```Java
输入：无
输出：无
方法：visitIincInsn
参数：对象位置，自增大小（int范围）
使用范例：
	mv.visitIincInsn(0, 200);
	mv.visitIincInsn(1, -40);
```
		
自增字节码是适用于int的字节码，在以下情境中会用到:

1. i++或i--或++i或--i

2. i+=x或i-=x

自增字节码可以使用负数。

### 按位且运算字节码：<i>x</i>and

x=i/l

```Java
输入：整数1 整数2
输出：按位且的整数
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(IAND);
```
		
这个字节码用于计算按位且操作（a&b）。

### 按位或运算字节码：<i>x</i>or

x=i/l

```Java
输入：整数1 整数2
输出：按位或的整数
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(LOR);
```
		
这个字节码用于计算按位或操作（a\|b）。

### 按位异或运算字节码：<i>x</i>xor

x=i/l

```Java
输入：整数1 整数2
输出：按位异或的整数
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(LXOR);
```
		
这个字节码用于计算按位或操作（a^b）。

同时，这个字节码还可以用于计算按位取反（这也是JVM的实现）：\~x=x^(-1)。

```Java
mv.visitVarInsn(ILOAD, 0);
mv.visitInsn(ICONST_M1);
mv.visitInsn(IXOR);
```

### 按位左移运算字节码：<i>x</i>shl

x=i/l

```Java
输入：整数 左移位数
输出：按位左移的整数
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(LSHL);
```
		
这个字节码用于计算按位左移操作（a<<b）。如果左移位数超过了32（int）或64（long）位，系统只会采取最低的5（int）或6（long）位进行左移操作。

### 按位右移运算字节码：<i>x</i>shr

x=i/l

```Java
输入：整数 右移位数
输出：按位右移的整数
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(ISHR);
```
		
这个字节码用于计算按位右移操作（a>>b）。如果右移位数超过了32（int）或64（long）位，系统只会采取最低的5（int）或6（long）位进行右移操作。

### 按位无符号右移运算字节码：<i>x</i>ushr

x=i/l

```Java
输入：整数 右移位数
输出：按位无符号右移的整数
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(LUSHR);
```

这个字节码用于计算按位无符号右移操作（a>>>b）。如果右移位数超过了32（int）或64（long）位，系统只会采取最低的5（int）或6（long）位进行无符号右移操作。

运算字节码说完之后，最后，来看看数字转换的字节码。

### 转换为float的字节码：<i>x</i>2f

x=i/l/d

```Java
输入：数字
输出：转换为float的数字
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(I2F);
```

转换为float采取了IEEE 754的取值规律，详见xadd。虽然对于int，float转换是由低级拓宽范围，但是由于float值不能取到所有int可表示的数字（float仅有24位精确数字，其他为指数和符号位），所以此转换仍然不精确。

### 转换为double的字节码：<i>x</i>2d

x=i/l/f

```Java
输入：数字
输出：转换为double的数字
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(F2D);
```
		
转换为double采取了IEEE 754的取值规律，详见xadd。对于int，这种转换是完全精确的。对于float，如果这个方法是FP-Strict，也就是采取了ACC_STRICT修饰（Java中的strictfp），这个计算就是精确的；如果不是，这个计算可能进行舍入。对于long，由于double值不能取到long表示的所有数字（double仅有53位精确数字，其他为指数和符号位），所以计算不精确。

### 转换为int的字节码：<i>x</i>2i

x=d/l/f

```Java
输入：数字
输出：转换为int的数字
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(F2I);
```
		
由于int在四种数字中级别最低，long转换为它时都有可能丢失精度（甚至符号位），float和double会使用IEEE 754“向零舍入”。特殊情况下，如果浮点数的NaN转换为int，值为0；如果浮点数超出int最大范围，则为相应符号下的最大值。

### 转换为long的字节码：<i>x</i>2l

x=i/f/d

```Java
输入：数字
输出：转换为long的数字
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(D2L);
```

由于long级别大于int，int转换为long不丢失精度。在浮点数下，long与int的转换规则类似。

### int转换为其他基本类型的字节码：i2<i>x</i>

x=b/c/s

```Java
输入：数字
输出：转换为byte/char/short的数字
方法：visitInsn
参数：无
使用范例：
	mv.visitInsn(I2B);
```
		
这三个字节码能分别将int缩小转换为byte（-128\~127）、short（-32768\~32767）和char（0\~65535）。由于是缩小变换，可能丢失精度甚至符号位。

### 下面是使用例子：计算平方和

Java代码如下：

```Java
public static double computeSquare2(int x1, int x2){
	return x1 * x1 + x2 * x2;
}
```
	
使用ASM写入，如下：

```Java
ClassWriter cw = new ClassWriter(ASM9);
cw.visit(V1_8, ACC_PUBLIC, "TestClass", null, "java/lang/Object", null);
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "computeSquare2", "(II)D", null, null);
mv.visitVarInsn(ILOAD, 0);
mv.visitInsn(DUP);
mv.visitInsn(IMUL);
mv.visitVarInsn(ILOAD, 1);
mv.visitInsn(DUP);
mv.visitInsn(IMUL);
mv.visitInsn(IADD);
mv.visitInsn(I2D);
mv.visitInsn(DRETURN);
mv.visitMaxs(3, 2);
mv.visitEnd();
cw.visitEnd();
```
	
将生成的类加载并调用，以参数100和21传入，结果为10441.0。

这篇博客到这里就结束了，下一期：Java ASM详解：MethodVisitor与Opcode（二）类、数组与调用

这篇文章一共讲了130个字节码呢~