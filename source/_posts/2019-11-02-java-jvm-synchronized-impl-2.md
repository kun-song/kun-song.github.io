---
title: synchronized 原理（二）：再谈重量级锁
date: 2019-11-02 16:15:05
tags:
  - synchronized
  - Monitor
categories: JVM
---

关于重量级锁的实现原理，前面有 3 篇相关文章：

* [synchronized 原理（一）：重量级锁](http://songkun.me/2019/10/29/2019-10-29-java-jvm-synchronized-impl/)：从 Javac 编译器、字节码和 JVM 执行引擎等方面介绍了重量级锁的实现原理；
* [JVM 管程介绍](http://songkun.me/2019/11/01/2019-11-01-java-jvm-object-monitor/)：介绍了重量级锁底层依赖的 Monitor 机制；
* [对象的内存布局](http://songkun.me/2019/10/13/2019-10-13-java-jvm-object-meomory-layout/)：介绍对象头中 Mark Word 部分的内容；

本文介绍重量级锁如何与 Monitor 关联起来。

<!-- more -->

两者关联的关键是对象头的 Mark Word 部分，Mark Word 存储 **对象自身的运行时数据**，包括 **锁状态**，当锁标志位为 10 时，表明该对象充当重量级锁，Mark Word 前 30 个比特存储 **指向重量级锁的指针**，根据 [HotSpot 文档：Synchronization and Object Locking](https://wiki.openjdk.java.net/display/HotSpot/Synchronization)：

>the class `ObjectMonitor` represents inflated locks.

inflated lock 即膨胀后的重量级锁，用 `ObjectMonitor` 类实现，结合 [JVM 管程介绍](http://songkun.me/2019/11/01/2019-11-01-java-jvm-object-monitor/) 可以推断，重量级锁即 Monitor。

因此锁标志位为 10 时，Mark Word 存储指向该对象关联的 Monitor 的指针，至于该 Monitor 是对象创建时随之创建，还是膨胀为重量级锁时创建，属于 JVM 实现细节。

对象的 Mark Word 有了指向 Monitor 的指针，则 `monitorenter` 和 `monitorexit` 指令可以根据对象引用获取对象关联的 Monitor，从而将加锁、解锁委托给 Monitor 实现。

总结：

1. 对象头中的 Mark Word 存储指向 Monitor 的指针；
2. `monitorenter` 和 `monitorexit` 提取参数对象关联的 Monitor，将加锁、解锁委托给它；

---

参考：

* [HotSpot 文档：Synchronization and Object Locking](https://wiki.openjdk.java.net/display/HotSpot/Synchronization)



