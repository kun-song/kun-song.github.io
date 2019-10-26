---
title: ThreadLocal 原理浅析
date: 2019-10-25 21:52:43
tags:
  - ThreadLocal
categories: Java
---

以前写过一篇 `ThreadLocal` 的源码分析（[源码分析 | ThreadLocal](http://songkun.me/2018/09/09/2018-09-09-source-code-reading-threadlocal/)），那时只是按部就班地阅读源码，有点只见树木，不见森林的局限，这次从更加宏观的角度分析下 `ThreadLocal` 的实现原理。

<!-- more -->

首先再次明确下 `ThreadLocal` 的用途：

>多个线程同时访问“共享变量”时，若该变量非线程安全，无法直接共享，则可为每个线程 **独立保存** 一个 **副本**，该副本相当于线程的局部变量，当前线程无法访问其他线程的线程本地变量。

使用场景有：

* 用作线程执行的 **上下文**，同一线程的各处都可访问，避免每个函数都显式传参：
  * Dubbo 中的上下文
  * 数据库连接池
* 将不安全的“共享”变量转换为线程本地变量，实现线程安全；

>关于第二点，该变量并非真正共享，而是可以不共享！

类似普通变量，线程本地变量需要能初始化、写入、读取：：

```Java
// 初始化
ThreadLocal<String> x = ThreadLocal.withInitial(() -> "Hello");
// 读取
x.get();
// 写入
x.set("world");
```

`ThreadLocal` 用法很简单，但初学时有很多疑问：

1. 静态的代码、动态的线程执行之间是如何关联的？
2. 如何实现线程“本地”的效果的？（每个线程一个副本，互不干扰）？

## 静态代码 vs 运行中的线程

线程是运行在 OS 上的一段代码，是动态的，对应到 Java 中的 `Thread` 实例，每个 `Thread` 实例都代表一个线程，它可能处于：

* NEW
* RUNNABLE
* BLOCKED/WAITING/TIMED_WAITING
* TERMINATED 
  
6 种状态之一，其中 RUNNABLE 状态对应 OS 的可运行状态 + 运行状态，处于该状态的线程我们可以认为正在 CPU 上执行。

通过 `Thread.currentThread()` 可以动态获取当前 **正在执行的线程** 对应的 `Thread` 实例，通过该实例，可以访问该线程的各种信息，从而把代码和运行中的线程关联起来，代码里可以对运行的线程做处理，比如：

```Java
Thread t = Thread.currentThread();
System.out.println(t.isInterrupted());  // 是否被中断
```

反过来，运行中的线程也可以修改自己的信息，即修改代表本线程的 `Thread` 实例的字段：

```Java
ThreadLocal<String> x = ThreadLocal.withInitial(() -> "Hello");
Thread t = Thread.currentThread();
t.xxmap.put(x, "hello");
```

实际上 `Thread` 几乎所有字段都是私有的，无法直接通过上面的方式修改。

## 如何实现“线程本地”？

线程本地变量，顾名思义是线程“私有”的，如何实现线程私有是理解 `ThreadLocal` 原理的最重要一步。

既然是线程“私有”变量，它应该保存在 `Thread` 类的 **私有字段** 中，线程运行时无法访问对方的私有字段，但 `ThreadLocal` 用法与这不沾边：

```Java
ThreadLocal<String> x = ThreadLocal.withInitial(() -> "Hello");
```

`x` 并没有保存在 `Thread` 的私有字段里啊，并且根本没有 `Thread` 出现好吧！

暂时认为由于某种魔法，`x` 中的 `String` 变量的确保存到了某个线程的私有字段中，但 `x` 可能被很多线程共享访问，每个线程是如何找到自己的那个副本的呢？

`Thread` 定义如下私有字段：

```Java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

线程本地变量 `x` 及其内容 `Hello` 就保存在这个私有的 `threadlocals` 中，它是一个 map，其元素类型为：

```Java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) { super(k); value = v; }
}
```

`Entry` 的 key 为 `ThreadLocal` 实例，value 为该实例的内容，即每个 `ThreadLocal` 实例对应一个 k-v 映射，只能保存一个线程本地变量。

`threadLocals` 作用域为 package，虽然定义在 `Thread` 类中，但对它的操作却定义在同一个包中的 `ThreadLocal` 中：

```Java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

* `t` 为当前线程，通过 `Thread.currentThread()` 获取；

现在回到 [源码分析 | ThreadLocal](http://songkun.me/2018/09/09/2018-09-09-source-code-reading-threadlocal/) 看 `get/set` 两个方法的实现就很明了：

1. 通过 `Thread t = Thread.currentThread()` 获取当前线程；
2. 获取当前线程的线程本地变量 map `ThreadLocalMap map = t.threadLocals`；
3. 从 `map` 中查询 or 插入 “`ThreadLocal` 实例 -> 内容” 的映射；

## 总结

1. 线程本地变量保存在一个 package priate 的 `Thread.threadLocals` 字段中，但 `Thread` 几乎不对它做任何操作，对它的初始化、插入、查询、删除等都由 `ThreadLocal` 实现。
2. `Thread.threadLocals` 类型为 `ThreadLocal.ThreadLocalMap`，是一个专为该场景设计的特殊 map；
3. `ThreadLocalMap` 的元素 `Entry` 是 `WeakReference` 的子类；
