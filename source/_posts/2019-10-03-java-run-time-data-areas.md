---
title: JVM 运行时数据区域
date: 2019-10-03 16:10:03
tags:
categories: JVM
---

>Java 8 运行时数据区域 [官方链接](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5)。

Java 源码被 Javac 编译为字节码，即 class 文件，class 文件有严格的格式限制（JVM 规范），不过大体上包含两类东西：

1. JVM 虚拟机指令
2. 数据

JVM 本身只是一个普通应用，它读取、执行 class 文件中的虚拟机指令，JVM 只是个普通进程，自然也有自己的 **内存**，由于执行 class 文件这件事非常复杂，JVM 将自己管理的内存按照用途划分为若干个不同的数据区域，不同区域有自己的用途，以及创建、销毁时间。

<!-- more -->

按照虚拟机规范，JVM 运行时所需数据分为两类：

1. 线程私有：
   1. 程序计数器（PC 寄存器）
   2. 虚拟机栈 & 本地方法栈
1. 线程共享：
   1. 堆
   2. 方法区

>以上区域都在 **内存** 中。

线程共享数据随 JVM 启动而生，随 JVM 停止而亡；线程独占数据随线程创建而生，随线程结束而亡。

## 线程私有

程序计数器、栈是线程私有数据，它们都与字节码的执行相关。

### 程序计数器

>If that method is not native, the pc register contains the **address** of the Java Virtual Machine **instruction** currently being executed.

程序计数器即 Program Counter Register，占用空间很小（是 JVM 规范唯一没有规定任何 OutOfMemoryError 的内存区域）可以视为 **当前线程** 所执行字节码的 **行号指示器**。

在虚拟机的 **概念模型** 中（仅概念模型，商业虚拟机可以使用其他更高效的方式实现），字节码解释器工作时通过改变 PC 寄存器的值来 **选取下一条要执行的字节码指令**，诸如分支、循环、跳转、异常处理、线程恢复等都依赖程序计数器实现。

Java 的线程模型基于具体操作系统平台的线程模型实现，大部分操作系统通过分时复用实现多线程，因此某一时刻，一个处理器核心只会执行 **一个线程** 中的指令，多个线程不断切换，轮流在内核上执行，为了线程切换后能恢复到正确的执行位置（字节码指令的行号），每个线程都需要有自己的程序计数器，以避免线程之间互相影响。

若线程正在执行的是：

1. Java 方法，则程序计数器保存 **正在执行的字节码指令的地址**；
2. Native 方法，则程序计数器为 **空**；

### 虚拟机栈 & 本地方法栈

>A Java Virtual Machine stack is analogous to the stack of a conventional language such as C: it holds **local variables** and **partial results**, and plays a part in method invocation and return.

大多数编程语言通过 **栈** 实现方法执行，一次方法调用到完成，对应一个 **栈帧** 的入栈到出栈。

Java 也不例外，并且根据方法类别，JVM 有两种栈：

1. 虚拟机栈：普通 Java 方法；
2. 本地方法栈：Native 方法；

>这两种栈作用、实现都非常类似，HotSpot 虚拟机甚至不区分两者，统一实现为虚拟机栈。

方法在执行的 **同时** 会创建一个 **栈帧**，栈帧包含方法执行所需的全部数据：

1. 局部变量表
2. 操作数栈
3. 动态链接
4. 方法返回地址

方法调用对应栈帧的入栈，方法执行完成对应栈帧的出栈。

栈帧中局部变量表包含方法在 **编译时** 可以确定的数据，包含 3 类数据类型：

1. 原始类型：对应 Java 的原始类型，如 `int` 等；
2. 引用类型：对应 Java 的引用类型，指向对象或数组；
3. `returnAddress` 类型：指向一条字节码指令的地址；

局部变量表的基本单位是槽（slot），除原始类型中的 `long` 和 `double` 占用两个 slot 外，其余类型均占一个 slot。

由于 **对象实例** 在堆上分配，局部变量表仅保存指向对象的引用，所以局部变量表的 **大小** 在编译时即可确定，运行时不会改变。

该区域的两种异常：

1. `StackOverflowError`：线程请求的 **栈深度** 大于虚拟机所允许的深度时抛出；
2. `OutOfMemoryError`：若栈允许动态扩展，且扩展时无法申请到足够内存时抛出；

## 线程共享

### 堆

>The heap is the run-time data area from which memory for all **class instances** and **arrays** is allocated.

堆包含“几乎”所有：

* 对象实例
* 数组

Java 应用中绝大部分数据以对象 or 数组形式存在，因此堆一般是运行时数据区域中 **最大** 的一块内存，也是垃圾收集器最关注的的地方，通常讲到垃圾收集时，就是指收集堆上的无用内存。

为了更快地分配内存 and 更好地回收内存（分代收集），堆可以细化为：

* 新生代
  * Eden 区
  * Survivor 区：包含两个大小相等的 from 区和 to 区；
* 老年代

>JVM 规范没有规定具体的垃圾收集算法，由具体实现自由选择。

异常：

* `OutOfMemoryError`：堆大小无法完成实例分配，且无法继续扩展时抛出；

### 方法区

>It stores per-class structures such as
>1. the run-time constant pool
>1. field and method data
>1. the **code** for methods and constructors, including the special methods ([§2.9](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.9)) used in class and instance initialization and interface initialization.

方法区存放每个类的详细信息，包括：

1. 运行时常量池
2. 字段元数据
3. 方法元数据
4. JIT 编译后的代码

注意，本文介绍的所有数据区域，包括方法区，都是 **逻辑概念**，具体的虚拟机可以自由实现。

例如在 HotSpot 1.7 之前，堆除了新生代、老年代外，还有一块永久代，永久代实现了方法区，两者等价，由于方法区存放在堆上，因此可以被垃圾收集器自动管理。

而诸如 JRocket 等 **没有永久代**，但它们肯定有方法区（毕竟是 JVM 规范规定的），只不过方法区在 JRocket 中是通过其他方式实现的而已。

方法区可以实现垃圾收集，也可以不实现，JVM 规范没有强制要求，如老 HotSpot 通过永久代实现方法区，取了个巧，免去为方法区单独实现垃圾收集的需要，直接复用普通堆内存的垃圾收集代码。

异常：

* `OutOfMemoryError`

#### 运行时常量池

>A run-time constant pool is a per-class or per-interface **run-time representation** of the **constant_pool table** in a class file ([§4.4](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4)).

`.class` 文件有非常严格的格式，其中有一个名为 `constant_pool` 的表，即常量池，其中包含 **编译时** 生成的：

1. 字面值：如 123、`“Hello”` 等
2. 符号引用
   * 类、接口的全限定名
   * 字段的名称、描述符
   * 方法的名称、描述符

`constant_pool` 表是 `.class` 文件中一个静态数据结构，当 JVM 加载 `.class` 文件后，会将 `constant_pool` 表的内容加载到方法区的运行时常量池中。

因此 `constant_pool` 表是常量池的静态表示，而方法区的 run-time constant pool 则是常量池的动态、运行时表示。

运行时常量池具备 **动态性**，包含的数据通常比 `constant_pool` 表更多，因为 Java 不强制要求常量只能在编译时产生，运行时同样可以产生常量，如 `String.intern()` 方法，即 run-time constant pool 是以下两者的集合：

1. 编译时产生的常量，即`constant_pool` 表；
2. 运行时动态产生的常量；

最后，因为每个类/接口对应一个 `.class` 文件，进而对应一个 `constant_pool` 表，因此每个类在方法区的数据结构中都包含属于自己的 run-time constant pool。

**疑问**：不同类之间的常量如何共享？？？

## 直接内存

直接内存（direct memory）并非 JVM 运行时数据区域的一部分，但 Java 可以“绕过” JVM，直接在操作系统级别进行内存的申请、释放，直接内存不受 JVM 垃圾收集保护。

例如可以通过各种 `Buffer` 向 OS 直接申请内存，Java 通过存储在堆上的 `DirectByteBuffer` 对象间接操作该内存，可以避免在 JVM 堆和 OS 内存之间来回拷贝数据，从而提升性能。

异常：

* `OutOfMemoryError`

