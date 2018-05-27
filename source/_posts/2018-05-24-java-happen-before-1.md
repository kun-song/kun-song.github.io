---
title: 基本概念 | 重排序、内存可见性、happen-before 关系
date: 2018-05-24 23:13:44
tags:
  - Java
  - 并发
  - JVM
categories: 技术
---

Java 内存模型其实是 **一组规则**，这些规则规定了线程对内存的访问排序、内存对线程的可见性等。

<!-- more -->

## 重排序

内存模型描述了程序的可能行为，Java 编译器可以 **任意** 生成字节码，只要保证字节码的行为与内存模型的  **预期结果** 保持一致。由于这个宽松的规定，编译器行为可以灵活多样，包括对操作进行重排序。

编译器对生成的字节码指令的重排序，主要是为了提升性能，重排序主要有：

1. 编译器生成的字节码指令的顺序，可以与源代码中的 **显然版本** 不同；
1. CPU 可以乱序 or 并行地执行指令；
1. 缓存可以改变写入内存的变量的次序；

## 内存可见性

现代计算机中，多个 CPU 共享 **同一内存** 非常常见，这会导致一个线程无法 **立即**（甚至永远）看到另一个线程操作产生的结果，因此 Java 内存模型规定了 JVM 的一种 **最小保证**：什么时候写入一个 **变量** 对其他线程可见。

多 CPU 体系结构中，每个 CPU 都有自己的缓存，并周期性与主内存协调一致。

若线程 A 写入一个变量值 V，随后线程 B 读取变量 V 的值，则下列场景中，B 读取的可能不是 A 写入的最新值：

* 执行线程 A 的 CPU 将变量 V 缓存到寄存器中；
* 执行线程 A 的 CPU 将变量 V 缓存到自己的缓存中，但还没有刷新同步到主内存中；
* 执行线程 B 的 CPU 的缓存中，有变量 V 的旧值；

## happen-before 关系

为了避免重排序、内存可见性的影响，JVM 规定了 happens-before 关系保证：

* 若线程 A 和线程 B 满足 happens-before 关系，则线程 A 执行操作的结果对线程 B 是可见的；
* 若两个操作未按 happens-before 排序，则 JVM 可以对其任意重排序；

happens-before 关系由很多法则保证，比如：

* Program order rule
  + Each action in a thread happens-before every action in that thread that comes later in the program order. 
  + 每个线程内，按顺序 happens-before
* Monitor lock rule
  + An unlock on a monitor lock happens-before every subsequent lock on that same monitor lock. 
  + 同一个监视器锁，解锁 happens-before 随后的所有加锁
* Volatile variable rule
  + A write to a volatile field happens-before every subsequent read of that same field. 
  + 同一个 `volatile` 字段，写操作 happens-before 随后的所有读操作
* Thread start rule
  + A call to Thread.start on a thread happens-before every action in the started thread.
  + 对 `Thread.start` 的调用 happens-before 该线程中的所有动作 
* Thread termination rule
  + Any action in a thread happens-before any other thread detects that thread has terminated, either by successfully return from Thread.join or by Thread.isAlive returning false. 
  + 线程 A 中的所有操作 happens-before 所有检测到 A 终止的线程
* Interruption rule
  + A thread calling interrupt on another thread happens-before the interrupted thread detects the interrupt (either by having InterruptedException tHRown, or invoking isInterrupted or interrupted). 
  + 线程 A 通过 `B.interrupt` 中断线程 B，该动作 happens-before 线程 B 检测到中断
* Finalizer rule
  + The end of a constructor for an object happens-before the start of the finalizer for that object. 
  + 构造器的结束 happens-before 该对象 finalizer 方法的开始
* Transitivity
  + If A happens-before B, and B happens-before C, then A happens-before C. 