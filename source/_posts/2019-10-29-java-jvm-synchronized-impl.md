---
title: synchronized 原理（一）：重量级锁
date: 2019-10-29 09:00:34
tags:
  - synchronized
categories: JVM
---

>`synchronized` 的锁分为重量级锁、轻量级锁、偏向锁 3 种，本文介绍重量级锁的实现原理。

`synchronized` 作为 Java 内置锁的默认实现，有两种用法：

* 同步方法（实例方法 + `static` 方法）
* 同步代码块

无论哪种用法，`synchronized` 的使用效果都一样，即同一时刻，只允许一个线程执行同步方法（or 同步代码块），有多个线程竞争时，其他线程 **阻塞** 等待。

>[Java 线程的生命周期](http://songkun.me/2019/10/20/2019-10-20-java-jvm-5-thread-states/) 中说过，只有 `synchronized` 能让线程进入 BLOCKED 状态。

<!-- more -->

在 JVM 字节码层面，同步方法、同步代码块的实现方式不同，需分开讲解。

## 同步代码块

同步代码块的加锁、解锁是通过 Javac 编译器实现的，就是一个 **语法糖** 而已，它需要保证无论代码块正常执行结束 or 抛出异常结束，都能正确释放锁，关键点为：

1. Javac 自动插入 `monitorenter` + `monitorerexit`；
2. Javac 自动插入异常表，保证即使抛出异常，也必然执行 `monitorexit`；

被 `synchronized` 修饰的代码块，被 Javac 编译成字节码后，会在代码块前后添加 `monitorenter` 和 `monitorexit` 两条字节码指令，例如：

```Java
public class Main {
    Object lock = new Object();

    public void f() {
        synchronized (this) {
            this.hashCode();
        }
    }
}
```

通过 `javap -c` 查看其编译后的字节码：

```
public void f();
  Code:
   0:   aload_0
   1:   dup
   2:   astore_1
   3:   monitorenter  // synchronized 入口
   4:   aload_0
   5:   invokevirtual   #4; //Method java/lang/Object.hashCode:()I
   8:   pop
   9:   aload_1
   10:  monitorexit  // synchronized 正常出口
   11:  goto    19
   14:  astore_2
   15:  aload_1
   16:  monitorexit  // synchronized 异常出口
   17:  aload_2
   18:  athrow
   19:  return
  Exception table:
   from   to  target type
     4    11    14   any
    14    17    14   any
```

### 正常结束

先看第一对：

```
0:   aload_0
1:   dup
2:   astore_1
3:   monitorenter  // synchronized 入口

9:   aload_1
10:  monitorexit  // synchronized 正常出口
```

同步代码块需要指定一个 **监视器对象**，本例的监视器为 `this`，因为 Java 字节码是 **基于栈** 的指令集，因此执行 `monitorenter` 时，该指令的操作数是 **操作数栈** 的栈顶元素，为了将 `this` 加载到操作数栈：

```
0:   aload_0   // 1. 将局部变量表 0 号 slot 的引用（this）加载到操作数栈
1:   dup       // 2. 复制栈顶元素，操作数栈中有两个 this
2:   astore_1  // 3. 将栈顶元素保存在局部变量表的 1 号 slot
```

每个方法 **局部变量表** 的第一个 slot 固定存放隐式的 `this` 参数，执行以上 3 条指令后，操作数栈的栈顶元素就是 `this` 引用。

`monitorexit` 同样需要 `this` 引用作为操作数：

```
9:   aload_1
10:  monitorexit  // synchronized 正常出口
```

因此加锁、解锁可以理解为：

```
monitorenter this
monitorexit this
```

### 异常结束

`synchronized` 需要保证，即使同步代码块中抛出异常，也能正确 **释放内置锁**，这是通过 Javac 自动插入异常表实现的：

```
Exception table:
 from   to  target type
   4    11    14   any
  14    17    14   any
```

第一个异常处理器的范围是 4-11 行字节码，异常类型为 any，对应代码块 `{}` 中的内容：

```
4:   aload_0
5:   invokevirtual   #4; //Method java/lang/Object.hashCode:()I
8:   pop
9:   aload_1
10:  monitorexit  // synchronized 正常出口
11:  goto    19
```

第二个异常处理器的范围是 14-17 行字节码，异常类型也是 any，对应第一个异常处理器的异常处理代码，对应 Java 的 `catch{}` 代码块：

```
14:  astore_2
15:  aload_1
16:  monitorexit  // synchronized 异常出口
17:  aload_2
```

两个异常处理器的跳转目标都是 `monitorexit`：

```
14:  astore_2
15:  aload_1
16:  monitorexit  // synchronized 异常出口
```

这样可以保证无论异常以何种姿势出现，最后都能执行 `monitorexit` 释放锁。

## 同步方法

同步方法不是通过插入 monitorenter、monitorexit 指令实现，并非语法糖，实现同步方法需要：

1. Javac 编译 `synchronized` 修饰的方法后，添加 ACC_SYNCHRONIZED 访问标识符；
2. JVM 保证 ACC_SYNCHRONIZED 方法执行前后可正确获取、释放锁；

把 `f` 改为同步方法：

```Java
public synchronized void f() {
    this.hashCode();
}
```

通过 `javap -c -v` 查看其字节码为：

```
public synchronized void f();
  descriptor: ()V
  flags: ACC_PUBLIC, ACC_SYNCHRONIZED  // 访问标志
  Code:
    stack=1, locals=1, args_size=1
       0: aload_0
       1: invokevirtual #2                  // Method java/lang/Object.hashCode:()I
       4: pop
       5: return
    LineNumberTable:
      line 3: 0
      line 4: 5
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0       6     0  this   LMain;
```

关键点在于方法的 **访问标志**（access_flags）：

```
flags: ACC_PUBLIC, ACC_SYNCHRONIZED
```

JVM 在执行方法时，若方法的访问标志里有 ACC_SYNCHRONIZED，则：

* 执行之前，当前线程必须先获取 monitor；
* 执行之后，无论正常 or 异常结束，当前线程必须释放 monitor；

虚拟机实现需要保证以上两点，应该是通过自动插入对 `monitorenter` 和 `monitorexit` 的调用实现的，其中 monitor 对象分别为：

* 同步实例方法：当前实例，即 `this`；
* 同步 `static` 方法：当前 `Class` 实例；

## 总结

1. 同步代码块仅依赖：
   1.  Javac 编译器为同步代码块添加 `monitorenter` + `monitorexit` + 异常处理器；
2. 同步方法依赖：
   1. Javac 为同步方法添加 ACC_SYNCHRONIZED 访问标志
   2. JVM 自动为携带 ACC_SYNCHRONZIED 的方法自动加锁、解锁

---

参考：

* 《深入理解 JVM 虚拟机》6.4.10 同步指令一节
