---
title: JEP 412 Foreign Function & Memory (FFM) API 外部函数与内存API
date: 2021-08-08
updated: 2021-08-30 11:30:21
category:
    - Java
tags: 
 - Java
 - JEP
comments: true
toc: true
---

这篇专栏翻译自<https://openjdk.java.net/jeps/412>，"JEP 412: Foreign Function & Memory API (Incubator)"，讲述了有关于Java 17中加入的FFM API。

***

## 前言

引入一个可以让Java程序与Java运行时以外的代码和数据进行交换的API。通过高效的调用外部函数（即JVM外部的代码），并且通过安全地访问外部内存（即不是由JVM管理的内存），这套API能让Java程序调用本地库和操作本地数据的同时避免JNI的脆弱性和不安全性。

## 目标

* 易用性 - 将Java本地接口（JNI）替换为优越的，纯Java开发的模型

* 性能 - 提供与现有 API（如 JNI 和 sun.misc.Unsafe）相媲美（如果不是更好的话）的性能

* 通用性 - 提供操作不同类型的外部内存（如：本地内存，永久内存和堆内存）的方法，并且随着时间的推移，去适应其他的平台（如：32bit x86）和除C以外的语言（如：C++,Fortran）编写的外部函数

* 安全性 - 在默认情况下禁用不安全的操作，仅在从应用程序开发人员或最终用户明确选择后才能禁用它们

## 非目标

* 在此 API 之上重新实现JNI，或以任何方式修改JNI

* 在此 API 之上重新实现传统 Java API，例如：sun.misc.Unsafe类

* 提供从本地代码头文件中自动生成 Java 代码的工具，或者

* 更改与本地库交互的 Java 应用程序的包装和部署方式（例如，通过多平台 JAR 文件）

## 内容

### 外部内存

存储在Java运行时之外的内存的数据被称为**堆外数据**(off-heap data)。"heap"（堆）是Java对象生存（对象生命周期）的地方，也是垃圾回收（Garbage Collector，GC）处理的地方。访问堆外数据对于Tensorflow、Ignite、Lucene和Netty等Java库的性能至关重要，这主要是因为这避免了由垃圾回收引起的成本和不可预测性，并且这也允许程序通过mmap等将文件映射入内存中进行数据结构的序列化和反序列化。但是，Java平台到今天没有为访问堆外数据提供令人满意的解决方案。

>ByteBuffer API（java.nio）允许创建直接缓冲区（direct buffer），这些缓冲区在堆外分配，但是它们的最大大小限制为2GB并且不能及时释放。这些和其他的限制都来源于一个事实：ByteBuffer API不仅被用于访问堆外内存，还被用于生产者/消费者之间批量数据的交换，如字符集的编码/解码和部分I/O操作。在这方面，它无法满足多年来提出的许多堆外内存增强请求。

>sun.misc.Unsafe API暴露了堆内内存的访问操作，这对堆外内存也适用。使用它很高效，因为它的内存操作被定义在HotSpot JVM内部并且会被JIT编译器优化。但是，因为它可以访问任何内存位置，使用它是危险的。这意味着一个Java程序可以通过访问一个已经释放的内存位置使JVM崩溃。因为这个和其他的原因，使用Unsafe是强烈不被推荐的。

>使用JNI调用本地库来访问堆外内存是可能的，但是因为它的效率开销（较高）而很少找到适用的地方：从Java到本地代码的速度要比直接访问内存的速度慢几个数量级，因为JNI方法调用并不能从常见的JIT优化（如内联）中获益。

总之，当访问堆外数据时，Java开发者就面临着两难的境地：他们是选择安全但效率不高的方式（ByteBuffer API）还是放弃安全转而选择性能（Unsafe API）？开发者需要的是一个Java支持的API，用于在JIT优化下从头到脚安全地访问堆外数据（即外部内存）。

### 外部函数

从Java 1.1开始，JNI就已经支持本地代码的调用（即外部函数），但是因为很多原因它并不适合。

>JNI涉及几个乏味的构件:Java API（本地方法）、源自Java API的C头文件（译注：即javah.exe的工作，当Java 10移除javah.exe后，这项工作由javac -h完成），以及调用感兴趣的本地库的C实现。Java开发人员必须跨多个工具链工作，以保持与平台相关的构件同步，当本地库快速发展时，这尤其繁重。

>JNI只能与以一些语言（通常为C/C++）进行交互，这些库使用了JVM在构建中使用的操作系统和CPU的约定。本地方法不能被用于去调用一个由不同约定的语言编写的函数。

>JNI没有协调Java类型系统和C类型系统。Java中的聚合数据是用对象表示的，但C中的聚合数据是用结构体表示的，因此传递给本地方法的任何Java对象都必须费力地由本地代码解包。例如，考虑一个Java中的记录（`record`，Java 16加入）类Person：将Person对象传递给本地方法将要求本地代码使用JNI的C API从对象中提取字段（例如，firstName和lastName）（译注：提取字段就是使用JNIEnv\*的函数）。结果是，Java开发者们有些时候会把他们的数据转变成一个单独的对象（如：一个字节数组或一个direct ByteBuffer），但更常见的是，因为通过JNI传递Java对象很慢，他们就使用Unsafe API去分配堆外内存并且以long的形式将内存地址传递给本地方法（译注：比如LWJGL）——可悲的是这使得Java代码变得不安全！

多年来，有许多框架填补JNI留下的空白，这其中包括`JNA`、`JNR`和`JavaCPP`。虽然这些框架通常被视为JNI的改进，但是情况依旧不理想，尤其是当与提供一流的本地代码交互的语言相比。比如，Python的ctypes包可以动态地将函数包装在本地库中而不用生成任何的粘合代码。其他语言，例如Rust，提供了可以从C/C++头文件中自动派生本地代码包装的工具。

总之，Java开发者应该有一个让他们能直接使用任何被认为对特定任务有用的本机库并且避免使用JNI带来的繁琐与沉闷的API。对于此的一个绝佳的抽象是方法句柄（`Method Handle`），它在Java 7被引入，用于支持在JVM上的快速动态语言（`invokedynamic`，inDy）。通过方法句柄公开本机代码将从根本上简化编写、构建和分发依赖于本机库的Java库的任务。此外，能够建模外部函数（即本机代码）和外部内存（即堆外数据）的API将为第三方本机交互框架提供坚实的基础。

### 描述

外部函数与内存API（Foreign Function & Memory API，下文简称为"FFM API"）定义了一系列类与接口以便于在库与应用程序中的客户端代码：

>分配外部内存（MemorySegment，MemoryAddress和SegmentAllocator）,
>操作和访问结构化外部内存（MemoryLayout，MemoryHandles和MemoryAccess），
>管理外部资源的生命周期（ResourceScope）和
>调用外部函数（SymbolLookup和CLinker）

FFM API定义在`jdk.incubator.foreign`模块下的`jdk.incubator.foreign`包内。

### 例子

下面是一个简单的使用`FFM API`的例子，Java代码获得了一个C库函数`radixsort`的方法句柄，然后用它来对Java数组中的四个字符串进行排序（一些细节被省略了）：

```Java
// 1. 在C库路径下寻找外部函数
MethodHandle radixSort = CLinker.getInstance().downcallHandle(
                             CLinker.systemLookup().lookup("radixsort"), ...);
// 2. 分配堆内内存储存4个字符串
String[] javaStrings   = { "mouse", "cat", "dog", "car" };
// 3. 分配堆外内存储存4个指针
MemorySegment offHeap  = MemorySegment.allocateNative(
                             MemoryLayout.ofSequence(javaStrings.length,
                                                     CLinker.C_POINTER), ...);
// 4. 将字符串从堆内复制到堆外
for (int i = 0; i < javaStrings.length; i++) {
    // 分配一个堆外的字符串， 然后储存一个指向它的指针
    MemorySegment cString = CLinker.toCString(javaStrings[i], newImplicitScope());
    MemoryAccess.setAddressAtIndex(offHeap, i, cString.address());
}
// 5. 通过调用外部函数将堆外数据排序
radixSort.invoke(offHeap.address(), javaStrings.length, MemoryAddress.NULL, '\0');
// 6. 将（已经排序后）的字符串数组从堆外复制到堆内
for (int i = 0; i < javaStrings.length; i++) {
    MemoryAddress cStringPtr = MemoryAccess.getAddressAtIndex(offHeap, i);
    javaStrings[i] = CLinker.toJavaStringRestricted(cStringPtr);
}
assert Arrays.equals(javaStrings, new String[] {"car", "cat", "dog", "mouse"});  // true
```

这段代码比任何使用JNI的解决方案都清晰得多，因为原本隐藏在本机方法调用后面的隐式转换和内存解引用现在直接用Java表示了。也可以使用现代Java语言特性；例如，流可以允许多个线程并行地在堆内和堆外内存之间复制数据。

### 内存段（Memory Segments）

内存段是对位于堆外或堆内的连续内存区域进行建模的抽象。内存段可以为

>本地段，在本地内存内从头开始分配（例如通过`malloc`），
>映射段，将映射包装在本地内存区域中（例如通过`mmap`），或者
>数组或缓冲区段，将现有的Java数组或字节缓冲区相关的内存分别包装

所有的内存段都提供了空间、时间和线程限制的保证，为了使内存解引用操作安全，这些保证都是强制的。例如，下面的代码在堆外分配了100个字节：

```Java
MemorySegment segment = MemorySegment.allocateNative(100, newImplicitScope());
```

段的*空间边界*决定了与段相关联的内存地址的范围。上面代码中段的边界由表示为`MemoryAddress`实例的*基础地址*b和以字节为单位的大小（100）定义，结果是地址范围从b到b + 99（包括b + 99）。

段的*时间边界*决定了段的生存期，也就是这个段什么时候会被释放。段的生存期和线程限制状态是通过`ResourceScope`抽象建模的，下面将对此进行讨论。上面代码中的资源作用域是一个新的*隐式*作用域，它确保当垃圾回收器认为`MemorySegment`对象不可达时才释放与此段相关的内存。隐式作用域还确保可以从多个线程访问内存段。

换句话说，上面的代码创建了一个行为与`allocateDirect`工厂分配的ByteBuffer的行为紧密匹配的段。FFM API还支持还支持确定性内存释放和其他线程限制选项，将在下面讨论。

### 解引用内存段

与段关联的内存解引用是通过获取变量句柄来实现的，它是Java 9中引入的数据访问抽象模型。特别地，段是用**内存访问变量句柄**来解引用的。这种类型的变量句柄使用一对访问坐标：

>以MemorySegment对象表示的坐标——也就是控制的内存要被解引用的段，和
>以long表示的坐标——也就是偏移量（offset），从段的基础地址到解引用开始的偏移量

内存访问变量句柄可以通过在`MemoryHandles`类中的工厂方法获取。例如，这段代码获取了可以将int写入本地内存段的内存访问变量句柄，并且使用它在连续的偏移下写入25个4字节的值（译注：指int为4字节）：

```Java
MemorySegment segment = MemorySegment.allocateNative(100, newImplicitScope());
VarHandle intHandle = MemoryHandles.varHandle(int.class, ByteOrder.nativeOrder());
for (int i = 0; i < 25; i++) {
    intHandle.set(segment, /* 偏移 */ i * 4, /* 要写入的数据 */ i);
}
```

更高级的访问用法可以通过使用MemoryHandles类提供的一个或多个组合子方法来组合内存访问变量句柄来表达。使用这些客户端可以，例如，对给定的内存访问变量句柄进行重排序，删除一个或多个坐标，或插入新的坐标。这允许创建接受一个或多个逻辑索引到一个在堆外内存区域的多维数组中的内存访问变量句柄。

为了使FFM API更容易访问，`MemoryAccess`类提供了静态访问器来解引用内存段，而不需要构造内存访问变量句柄。例如，有一个访问器可以在给定偏移量的段中设置一个int值，允许上面的代码简化为：

```Java
MemorySegment segment = MemorySegment.allocateNative(100, newImplicitScope());
for (int i = 0; i < 25; i++) {
    MemoryAccess.setIntAtOffset(segment, i * 4, i);
}
```

### 内存布局（Memory Layouts）

为了减少对内存布局的繁琐计算(例如，上面例子中的i * 4)， `MemoryLayout`可以用更声明式的方式来描述内存段的内容。例如，上面例子中需要的本地内存段的布局可以用以下方式描述：

```Java
SequenceLayout intArrayLayout
    = MemoryLayout.sequenceLayout(25,
        MemoryLayout.valueLayout(32, ByteOrder.nativeOrder()));
```

这将创建一个*序列内存布局*（`sequence memory layout`），内部由重复了25次的32比特*值布局*（一个描述了单一32字节值的布局）构成。给定一个内存布局，我们可以避免在代码中计算偏移量，并简化内存分配和创建内存访问变量句柄：

```Java
MemorySegment segment = MemorySegment.allocateNative(intArrayLayout, newImplicitScope());
VarHandle indexedElementHandle =
    intArrayLayout.varHandle(int.class, PathElement.sequenceElement());
for (int i = 0; i < intArrayLayout.elementCount().getAsLong(); i++) {
    indexedElementHandle.set(segment, (long) i, i);
}
```

`intArrayLayout`对象通过创建布局路径来驱动内存访问变量句柄的创建，该路径用于从复杂布局表达式中选择嵌套布局。intArrayLayout对象也驱动了本地内存段的分配，这个内存段基于来自于布局的大小和对齐信息。在之前的例子中的循环常数，也就是25，已经被序列布局的元素数量所替代。

### 资源作用域（Resource Scopes）

在前面的例子中看到的所有内存段都使用了非确定性的释放：一旦内存段实例变得不可达，垃圾收集器就会释放与这些段相关的内存。我们说这样的段是**隐式释放**的。

在某些情况下，客户端可能希望控制何时发生内存释放。试想，例如，使用`MemorySegment::map`从一个文件中映射出一个很大的内存段。客户端可能更喜欢在段不再需要时释放（即取消映射）与段相关的内存，而不是等待垃圾收集器这样做，因为等待可能会对应用程序的性能产生不利影响。

内存段支持通过资源作用域的确定性释放。资源作用域对与一个或多个资源（如内存段）相关联的生命周期进行建模。新创建的资源作用域处于*活动*状态，这意味着可以安全地访问它管理的所有资源。在客户端请求时，可以*关闭*资源作用域，这意味着不再允许访问由该作用域管理的资源。因为ResourceScope类实现了`AutoClosable`接口，所以它可以使用`try-with-resource`语句：

```Java
try (ResourceScope scope = ResourceScope.newConfinedScope()) {
    MemorySegment s1 = MemorySegment.map(Path.of("someFile"), 0, 100000,
                                         MapMode.READ_WRITE, scope);
    MemorySegment s2 = MemorySegment.allocateNative(100, scope);
    ...
} // 这两个内存段到这里被释放
```

这段代码创建了一个受限（`confined`）的资源作用域，并将其用于创建两个段：映射段（s1）和本地段（s2）。这两个段的生命周期与资源作用域的生命周期相关联，因此在try-with-resources语句完成后访问段（例如，使用内存访问变量句柄对它们进行解引用）将导致抛出一个运行时异常。

除了管理内存段的生命周期外，资源作用域还可以作为一种方法来控制哪些线程可以访问内存段。受限资源作用域只允许创建作用域的线程的访问，而共享资源作用域允许从任何线程访问。

资源作用域，无论是受限的还是共享的，都可能与`java.lang.ref.Cleaner`对象相关联，该对象负责执行隐式释放，以防在客户端调用`close`方法之前，资源作用域对象变得不可达。

一些称为隐式资源作用域的资源作用域不支持显式释放——调用close将失败。隐式资源作用域总是使用Cleaner来管理它们的资源。隐式作用域可以使用`ResourceScope::newImplicitScope`工厂创建，如前面的示例所示。

### 段分配器（Segment Allocators）

当客户端使用堆外内存时，内存分配通常是一个瓶颈。FFM API包括一个`SegmentAllocator`抽象模型，它定义了分配和初始化内存段的操作。段分配器是通过SegmentAllocator接口中的工厂获得的。例如，下面的代码创建了一个*基于区域*（`arena-based`）的分配器，并使用它来分配一个内容是从Java int数组初始化的段：

```Java
try (ResourceScope scope = ResourceScope.newConfinedScope()) {
    SegmentAllocator allocator = SegmentAllocator.arenaAllocator(scope);
    for (int i = 0 ; i < 100 ; i++) {
        MemorySegment s = allocator.allocateArray(C_INT, new int[] { 1, 2, 3, 4, 5 });
        ...
    }
    ...
} // 所有分配的内存在此处被释放
```

这段代码创建一个受限的资源范围，然后创建与该范围相关联的*无边界区域分配器*（`unbounded arena allocator`）。这个分配器将分配特定大小的内存块，并通过返回预先分配的内存块的不同片（译注：也就是分配器先分配一定大小的块后，用户要求内存时按用户需求在内存块中取出相当长度的内存切片）来响应分配请求。如果一个内存块没有足够的空间来容纳一个新的分配请求，那么就分配一个新的内存块。如果与区域分配器相关联的资源作用域被关闭，所有与分配器创建的段相关联的内存（例如，在for循环体中）都会被以原子方式释放。这种用法结合了ResourceScope抽象提供的确定性释放的优点，以及更灵活和可伸缩的分配方案。在编写管理大量堆外内存段的代码时，它非常有用。

### 不安全的内存段

到目前为止，我们已经看到了内存段、内存地址和内存布局。解引用操作只能在内存段上进行。由于内存段具有空间和时间边界，Java运行时总是可以确保与给定段相关联的内存被安全解引用。然而，在某些情况下，客户端可能只有MemoryAddress实例，这在与本机代码交互时经常发生。由于Java运行时无法知道与内存地址相关的空间和时间边界，因此FFM API禁止直接解引用内存地址。

为了解引用内存地址，客户端有两种选择：

>如果已知地址位于一个内存段，客户端可以通过`MemoryAddress::segmentOffset`进行**重新基准**（`rebase`）操作。重新基准操作会重新定义地址相对于段的基本地址的偏移量，以产生一个新的可以应用于现有段上的偏移量——然后可以安全地对该段解引用。

>或者，如果没有这样的段存在，那么客户端可以使用`MemoryAddress::asSegment`工厂不安全地创建一个。这个工厂有效地将新的空间和时间边界附加到一个原始的内存地址，以便允许解引用操作。该工厂返回的内存段是不安全的：一个原始内存地址可能与一个10字节长的内存区域相关联，但客户端可能意外地高估了该区域的大小，并创建了一个100字节长的不安全内存段。这可能会导致稍后试图对与不安全段关联的内存区域边界之外的内存的解引用，这可能会导致JVM崩溃，或者更糟的是，导致在无形中的内存损坏。因此，创建不安全的段被视为**受限操作**，默认情况下是禁用的（参见下面的详细内容）。

### 寻找外部函数

任何对外部函数的支持的第一个组成部分都是加载本地库的机制。在JNI中，这是通过`System::loadLibrary`和`System::load`方法完成的，它们在内部映射到对`dlopen`或其等效函数的调用。使用这些方法加载的库总是与类加载器（即调用`System`方法的类加载器）相关联。库和类加载器之间的关联是至关重要的，因为它管理装入的库的生命周期：只有当类加载器不再可访问时，它的所有库才能被安全卸载。

FFM API没有提供加载本地库的新方法。开发者使用System::loadLibrary和System::load方法来加载将通过FFM API调用的本地库。库和类加载器之间的关联被保留，因此库将以与JNI相同的可预测方式卸载。

与JNI不同，FFM API提供了在加载的库中查找给定标识地址的功能。这种由`SymbolLookup`对象表示的功能对于将Java代码链接到外部函数至关重要（参见下面）。有两种方法可以获得SymbolLookup对象：

* `SymbolLookup::loaderLookup`返回一个包括本加载器内加载的所有库内部的标识的查找器

* `CLinker::systemLookup`返回一个特定于平台的标识查找器，它能查找标准C库内的标识

给定一个标识查找器，客户端可以使用`SymbolLookup::lookup(String)`方法找到一个外部函数。如果指定的函数出现在标识查找器所包括的标识中，则该方法返回指向函数入口点的MemoryAddress。例如，下面的代码加载OpenGL库（使它与当前类加载器相关联），并找到它的`glGetString`函数的地址：

```Java
System.loadLibrary("GL");
SymbolLookup loaderLookup = SymbolLookup.loaderLookup();
MemoryAddress clangVersion = loaderLookup.lookup("glGetString").get();
```

### 将Java代码链接到外部函数

`CLinker`接口是Java代码与本地代码交互的核心。虽然CLinker专注于提供Java和C库之间的互操作，但接口中的概念已经足够通用，可以在未来支持其他非Java语言。该接口支持向下调用（`downcall`，从Java代码调用本地代码）和向上调用（`upcall`，从本地代码调用回Java代码）。

```Java
interface CLinker {
    MethodHandle downcallHandle(MemoryAddress func,
                                MethodType type,
                                FunctionDescriptor function);
    MemoryAddress upcallStub(MethodHandle target,
                             FunctionDescriptor function,
                             ResourceScope scope);
}
```

对于向下调用，`downcallHandle`方法接受外部函数的地址——通常是从库查找中获得的MemoryAddress——并将外部函数作为向下调用方法句柄公开。稍后，Java代码通过调用`invokeExact`方法调用downcall方法句柄，然后运行外部函数。传递给方法句柄的invokeExact方法的任何参数都会传递给外部函数。

对于上行调用，`upcallStub`方法接受一个方法句柄——通常是指一个Java方法句柄，而不是下行调用方法句柄——并将其转换为内存地址。稍后，当Java代码调用downcall方法句柄时，将内存地址作为参数传递。实际上，内存地址充当函数指针。（欲了解更多关于upcall的信息，请参阅下面）

假设我们想从Java向下调用定义在C标准库中的`strlen`函数：

```C
size_t strlen(const char *s);
```

一个暴露strlen的向下调用方法句柄可以像下面这样获取（关于`MethodType`和`FunctionDescriptor`的细节将会简短介绍）：

```Java
MethodHandle strlen = CLinker.getInstance().downcallHandle(
    CLinker.systemLookup().lookup("strlen").get(),
    MethodType.methodType(long.class, MemoryAddress.class),
    FunctionDescriptor.of(C_LONG, C_POINTER)
);
```

调用向下调用方法句柄会执行strlen并且让结果在Java端可见。对于strlen的参数，我们使用一个helper方法将Java的字符串转变为堆外内存段并且传递这个段的地址：

```Java
MemorySegment str = CLinker.toCString("Hello", newImplicitScope());
long len          = strlen.invokeExact(str.address());  // 5
```

方法句柄在公开外部函数时工作得很好，因为JVM已经优化了方法句柄的调用，一直优化到本地代码。当方法句柄引用类文件中的方法时，调用方法句柄通常会导致目标方法被JIT编译；随后，JVM通过将控制转移到为目标方法生成的汇编代码来解释调用MethodHandle::invokeExact的Java字节码。因此，调用传统方法句柄已经几乎是外部调用；以C库中的函数为目标的downcall方法句柄只是一种更外部的方法句柄形式。方法句柄还具有一个名为签名多态性的属性，该属性允许基本类型参数的非装箱传入（译注：就是直接传int而不是Integer避免装箱/拆箱操作）。总之，方法句柄让CLinker以一种自然、有效和可扩展的方式公开外部函数。

### 在Java中描述C类型

为了创建向下调用方法句柄，FFM API需要客户端提供对于目标C函数的两种签名：使用*非透明*的Java对象（MemoryAccess和MemorySegment）的高级别签名和使用*透明*的Java对象（MemoryLayout）的低级别签名。依次取每个签名：

>高级别签名，即MethodType，用作向下调用方法句柄的类型。每个方法句柄都是强类型的，这意味着可以传递给它的invokeExact方法的参数的数量和类型是严格的。例如，为接受一个MemoryAddress参数而创建的方法句柄不能通过invokeExact(MemoryAddress, MemoryAddress)或通过invokeExact("Hello")调用。因此，MethodType描述了客户端在调用向下调用方法句柄时必须使用的Java签名。实际上，它是C函数的Java视图。

>低级别签名，即FunctionDescriptor，包含MemoryLayout对象。这使CLinker能够精确地理解C函数的参数，以便它能够正确地安排它们，如下所述。客户端通常有MemoryLayout对象，以便解引用外部内存中的数据，这样的对象可以在这里作为外部函数签名重用。

例如，为接受int值并返回long值的C函数获取向下调用方法句柄时，downcallHandle方法需要以下MethodType和FunctionDescriptor参数：

```Java
MethodType mtype         = MethodType.methodType(long.class, int.class);
FunctionDescriptor fdesc = FunctionDescriptor.of(C_LONG, C_INT);
```

>这个例子的目标系统是Linux/x64和macOS/x64，其中Java类型long和int分别与预定义的CLinker布局C_LONG和C_INT关联。Java类型与内存布局的关联因平台而异：例如，在Windows/x64上，Java long与C_LONG_LONG布局相关联

>译注：这里的原因是C中long的位数取决于系统，而long long为确定64位；在Java中，int确定32位而long为64位，为了确保数据的对齐需要调整布局

另一个例子，获取一个带有指针的void C函数的向下调用方法句柄需要以下MethodType和FunctionDescriptor：

```Java
MethodType mtype         = MethodType.methodType(void.class, MemoryAddress.class);
FunctionDescriptor fdesc = FunctionDescriptor.ofVoid(C_POINTER);
```

>C语言中的所有指针类型在Java中都表示为MemoryAddress对象，对应的布局是C_POINTER，其大小取决于当前平台。客户端不会区分int\*和char\*\*，因为传递给CLinker的Java类型和内存布局包含足够的信息来正确地将Java参数传递给C函数

最后，与JNI不同的是，CLinker支持将结构化数据传递给外部函数。获取一个接受struct的无返回值C函数的向下调用方法句柄需要以下MethodType和FunctionDescriptor：

```Java
MethodType mtype         = MethodType.methodType(void.class, MemorySegment.class);
MemoryLayout SYSTEMTIME  = MemoryLayout.ofStruct(
  C_SHORT.withName("wYear"),      C_SHORT.withName("wMonth"),
  C_SHORT.withName("wDayOfWeek"), C_SHORT.withName("wDay"),
  C_SHORT.withName("wHour"),      C_SHORT.withName("wMinute"),
  C_SHORT.withName("wSecond"),    C_SHORT.withName("wMilliseconds")
);
FunctionDescriptor fdesc = FunctionDescriptor.ofVoid(SYSTEMTIME);
```

>对于高级别的MethodType签名，Java客户端总是使用不透明的类型MemorySegment，其中C函数需要一个按值传递的struct。对于低级别的FunctionDescriptor签名，与C结构类型相关联的内存布局必须是一个复合布局，它定义了C的struct中所有字段的子布局，包括可能由本地编译器插入的填充

如果C函数返回由低级别签名表示的按值struct，则必须在堆外分配一个新的内存段并返回给Java客户端。为了实现这一点，downcallHandle返回的方法句柄需要一个额外的SegmentAllocator参数，FFM API使用该参数分配内存段来保存C函数返回的struct。

### 为C函数打包Java参数

不同语言之间的交互操作需要一个调用约定来指定一种语言中的代码如何调用另一种语言中的函数、如何传递参数以及如何接收任何结果。CLinker实现具有一些"开箱即用"的调用约定的知识：Linux/x64、Linux/AArch64、macOS/x64和Windows/x64。CLinker是用Java编写的，维护和扩展起来要比JNI容易得多，JNI的调用约定是硬连接到HotSpot的C++代码中的（译注：JNI的调用约定即JNIEnv\*）。

考虑上面显示的SYSTEMTIME结构和布局的函数描述符（FunctionDescriptor）。根据运行JVM的操作系统和CPU的调用约定，当使用MemorySegment参数调用向下调用方法句柄时，CLinker使用函数描述符来推断结构体的字段应该如何传递给C函数。对于一个调用约定，CLinker可以安排分解传入的内存段，使用通用CPU寄存器传递前四个字段，并在C堆栈上传递其余字段。对于不同的调用约定，CLinker可以安排FFM API通过分配一个内存区域来间接传递结构体，将传入内存段的内容批量复制到该区域，并将指向该内存区域的指针传递给C函数。这种最低层次的参数打包是在幕后进行的，不需要任何客户端代码的监督。

### 向上调用

有时，将Java代码作为函数指针传递给某个外部函数是很有用的。我们可以通过使用对上行调用的CLinker支持来实现这一点。在本节中，我们将逐块构建一个更复杂的示例，该示例演示了CLinker的全部功能，以及代码和数据跨Java/本地边界的完全双向互操作。

考虑标准C库中定义的以下函数：

```C
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *, const void *));
```

为了从Java端调用`qsort`，我们首先需要创建向下调用方法句柄：

```Java
MethodHandle qsort = CLinker.getInstance().downcallHandle(
    CLinker.systemLookup().lookup("qsort").get(),
    MethodType.methodType(void.class, MemoryAddress.class, long.class,
                          long.class, MemoryAddress.class),
    FunctionDescriptor.ofVoid(C_POINTER, C_LONG, C_LONG, C_POINTER)
);
```

和前面一样，我们使用C_LONG和long.class来映射C size_t类型，并且在第一个指针形式参数（数组指针）和最后一个形式参数（函数指针）上使用MemoryAddress.class。

qsort使用作为函数指针传递的自定义比较器函数`compar`对数组的内容进行排序。因此，要调用向下调用方法句柄，我们需要一个函数指针作为最后一个参数传递给方法句柄的invokeExact方法。CLinker::upcallStub通过使用现有的方法句柄帮助我们创建函数指针，如下所示。

首先，我们在Java中编写一个静态方法来比较两个long值，间接表示为MemoryAddress对象：

```Java
class Qsort {
    static int qsortCompare(MemoryAddress addr1, MemoryAddress addr2) {
        return MemoryAccess.getIntAtOffset(MemorySegment.globalNativeSegment(),
                                           addr1.toRawLongValue()) -
               MemoryAccess.getIntAtOffset(MemorySegment.globalNativeSegment(),
                                           addr2.toRawLongValue());
    }
}
```

接着，我们创建一个指向Java比较方法的MethodHandle：

```Java
MethodHandle comparHandle
    = MethodHandles.lookup()
                   .findStatic(Qsort.class, "qsortCompare",
                               MethodType.methodType(int.class,
                                                     MemoryAddress.class,
                                                     MemoryAddress.class));
```

之后，现在我们有了Java比较器的方法句柄，我们可以使用CLinker::upcallStub创建函数指针。就像向下调用一样，我们使用CLinker类中的布局来描述函数指针的签名：

```Java
MemoryAddress comparFunc =
  CLinker.getInstance().upcallStub(comparHandle,
                                   FunctionDescriptor.of(C_INT,
                                                         C_POINTER,
                                                         C_POINTER),
                                   newImplicitScope());
);
```

我们终于有了一个内存地址，`comparFunc`，它指向一个方法存根，可以用来调用我们的Java比较方法，所以现在我们有了调用qsort向下调用句柄所需的所有东西：

```Java
MemorySegment array = MemorySegment.allocateNative(4 * 10, newImplicitScope());
array.copyFrom(MemorySegment.ofArray(new int[] { 0, 9, 3, 4, 6, 5, 1, 8, 2, 7 }));
qsort.invokeExact(array.address(), 10L, 4L, comparFunc);
int[] sorted = array.toIntArray(); // [ 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 ]
```

这段代码创建了一个堆外数组，将Java数组的内容复制到其中，然后将数组连同我们从CLinker获得的比较器函数（指针）传递给qsort句柄。调用之后，堆外数组的内容将根据我们用Java编写的比较器函数进行排序。然后从段中提取一个新的Java数组，其中包含已排序的元素。

### 安全

基本上，Java代码和本机代码之间的任何交互都可能危及Java平台的完整性。链接到预编译库中的C函数本质上是不可靠的，因为Java运行时不能保证函数的签名符合Java代码的期望，甚至不能保证C库中的标识是真正的函数。此外，如果链接了一个合适的函数，实际上调用该函数可能会导致如分段错误的底层故障，最终导致VM崩溃。Java运行时无法阻止此类故障，Java代码也无法捕获此类故障。

使用JNI函数的本地代码尤其危险。这样的代码可以在没有命令行标志（例如`--add-open`）的情况下，通过使用`getStaticField`和`callVirtualMethod`等函数访问JDK内部。它还可以在final字段初始化很久之后更改它们的值。它允许本地代码绕过应用于Java代码的检查，这会破坏JDK中的每个边界和假设。换句话说，JNI本质上就是不安全的。

JNI不能被禁用，因此无法确保Java代码不会调用使用危险的JNI函数的本地代码。这是对平台完整性的一种风险，应用程序开发人员和最终用户几乎看不到这种风险，因为这些函数99%的使用通常来自夹在应用程序和JDK之间的第三、第四和第五方库。

大多数FFM API的设计是安全的。过去需要使用JNI和本地代码的许多场景都可以通过调用不会危及Java平台的FFM API中的方法来实现。例如，JNI的一个主要用例——灵活的内存分配——是由一个简单的方法MemorySegment::allocateNative支持的，该方法不涉及本机代码，并且总是返回由Java运行时管理的内存。一般来说，使用FFM API的Java代码不会使JVM崩溃。

然而，FFM API的一部分本身就是不安全的。当与CLinker交互时，Java代码可以通过指定与底层C函数不兼容的参数类型来请求向下调用方法句柄。在Java中调用向下调用方法句柄会导致与在JNI中调用本机方法时相同的结果——VM崩溃或未定义的行为。FFM API也可以产生不安全的段，即内存段的空间和时间边界是用户提供的，这种段不能由Java运行时验证（参见上文的MemoryAddress::asSegment）。

FFM API中的不安全方法不会带来与JNI函数相同的风险：例如，它们不能更改Java对象中的final字段的值。另一方面，FFM API中的不安全方法很容易从Java代码中调用。由于这个原因，FFM API中不安全方法的使用受到限制：默认情况下，不安全方法的访问是禁用的，调用这些方法会抛出一个IllegalAccessException异常。要使某些模块M中的代码能够访问不安全的方法，请在命令行中指定`java --enable-native-access=M`。（在以逗号分隔的列表中指定多个模块；指定`ALL-UNNAMED`以允许类路径上的所有代码访问不安全方法）FFM API的大多数方法都是安全的，Java代码可以使用这些方法，不管是否给出了--enable-native-access。

我们在这里不建议限制JNI的任何方面。在Java中仍然可以调用本地方法，本地代码也可以调用不安全的JNI函数。然而，在未来的版本中，我们可能会以某种方式限制JNI。例如，不安全的JNI函数（如`newDirectByteBuffer`）可能会在默认情况下被禁用，就像FFM API中的不安全方法一样。更广泛地说，JNI机制是如此的危险，以至于我们希望库在安全和不安全的操作中偏向于纯Java的FFM API，这样我们就可以在默认情况下禁用所有JNI。这与使平台成为“开箱即用”的安全平台的更广泛的Java路线图一致，要求终端用户选择不安全的行为，如破坏强封装或链接到未知代码。

我们不建议以任何方式去修改`sun.misc.Unsafe`。FFM API对堆外内存的支持是对sun.misc.Unsafe中的`malloc`和`free`，即`allocateMemory`, `setMemory`, `copyMemory`，和`freeMemory`的一个很好的替代方案。我们希望需要非堆存储的库和应用程序采用FFM API，以便及时地弃用并最终删除这些sun.misc.Unsafe方法。

### 选择

继续使用java.nio.ByteBuffer，sun.misc.Unsafe，JNI和其他第三方框架。

### 风险和假设

创建一个API以既安全又高效的方式访问外部内存是一项艰巨的任务。由于前几节中描述的空间和时间检查需要在每次访问时执行，因此JIT编译器能够优化这些检查是至关重要的，例如，将它们提升到热循环之外。JIT实现可能需要做一些工作，以确保API的使用与ByteBuffer和Unsafe等现有API的使用一样有效和可优化。JIT实现还需要确保从API中检索到的本地方法句柄的使用至少与使用现有JNI本地方法一样有效和可优化。

### 依赖

>外部函数和内存API可以用来访问非易失性内存，已经可以通过JEP 352（非易失性映射字节缓冲区，Non-Volatile Mapped Byte Buffers，Java 14引入）用一种更通用和更有效的方式访问

>这里描述的工作可能会使后续工作能够提供一个工具，jextract，它从给定本地库的头文件开始，机械地生成与该库交互操作所需的本机方法句柄。这将进一步减少使用Java本地库的开销