---
title: JVM 管程介绍
date: 2019-11-01 23:15:51
tags:
  - synchronized
  - Monitor
  - Object.wait
  - Object.notify
  - Object.notifyAll
categories: JVM
---

`synchronized` 加锁、解锁是通过 `monitorenter`、`monitorexit` 这两条字节码指令实现的，这两个字节码需要指定一个 **对象引用** 作为参数，它们底层基于 Monitor 实现加锁、解锁语义。

<!-- more -->

## 名字澄清

根据 Java 语言规范：

>The most basic of these methods is synchronization, which is implemented using **monitors**. Each object in Java is associated with a monitor, which a thread can lock or unlock.

即 `synchronized` 是基于 monitor 实现的，加锁、解锁操作最终都落到对 monitor 的加锁、解锁上，并且 **每个对象** 都关联一个 monitor，因此每个对象都可以作为 `monitorenter` 和 `monitorexit` 的参数，这两条指令使用的是参数对象关联的 monitor，而非对象本身。

monitor 也叫做管程 or 监视器，是加锁、解锁的最终实现之处，在 HotSpot 虚拟机中，使用 C++ 通过 `ObjectMonitor` 对象实现 Java 中的 monitor 语义。

网上也经常遇到 **内置锁** 的说法，内置锁和 monitor 是什么关系呢，还是看 [官方资料](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)：

>Synchronization is built around an internal entity known as the **intrinsic lock** or **monitor lock**. (The API specification often refers to this entity simply as a "**monitor**.")
>...
>**Every object** has an intrinsic lock associated with it.

内置锁即 intrinsic lock，也叫 monitor lock，简称 monitor。

因此以下名字实际都是同一概念：

* 内置锁
* 监视器锁
* 监视器
* 管程
* Monitor

理解 Monitor 对于理解 Java 语言中的同步是非常重要的，它是以下语义的实现之处：

1. 多线程互斥访问
2. 内置的条件队列
3. 内置锁加锁、解锁的 happens-before 关系

## 基本概念

根据维基百科，管程有两种理解方式：

* 同步机制（synchronization construct）
* 线程安全的类、对象 or 模块（通过 mutex 实现）

### 同步机制

管程作为一种同步机制，可赋予线程以下能力：

1. mutual exclusion
2. the ability to **wait** (block) for a certain **condition** to become true

线程进入 BLOCKED 状态等待 **条件变量** 后，还需要有机制能够 **唤醒** 它们：

* Monitors also have a mechanism for **signaling** other threads that their **condition** has been met.

因此管程有 3 大能力：

1. 互斥访问
2. 线程协作：等待 + 唤醒

线程如何实现 **互斥**，如何在条件变量不满足时进行 **等待**，如何在条件变量满足时被 **唤醒** 以继续运行，这些都是管程实现需要解决的问题，对于用户而言，管程就是一个提供 3 大功能的黑盒。

>基于等待-唤醒的线程协作最典型的例子是生产者-消费者模型：
>* 生产者 **等待** **队列非满** 条件，并 **唤醒** 等待 **队列非空** 条件的消费者；
>* 消费者 **等待** **队列非空** 条件，并 **唤醒** 等待 **队列非满** 条件的生产者；

### 线程安全的类

除了将管程视为 *抽象* 的同步机制外，还可以将管程视为该机制的 *具体实现*，表现为线程安全的类 or 对象，它同样提供 3 大能力：

1. 对该类方法的互斥访问；
2. 等待 & 唤醒

## Monitor 概念模型

使用 `synchronized` 时，就是使用 Monitor，`synchronized` 是一个非常简洁的接口，让我们无需关注如何实现互斥、等待、唤醒等，JVM 封装了这些枯燥、容易出错的实现细节。

对应管程的 3 大能力，JVM 中的 Monitor 提供：

* 互斥访问：接口为 `synchronized`
* 线程协作：接口为 `Object.wait` 和 `Object.notify/notifyAll`

针对 Monitor 的操作有：

1. 获取 Monitor
2. 持有 Monitor
3. 释放 Monitor

每个对象都与一个 monitor 关联，每个 monitor 都包含：

1. entry set：排队获取 Monitor 的队列；
2. wait set：等待 *特定条件* 为真的队列；

### 互斥访问：entry set

互斥访问借助 entry set 实现：

1. 线程 t 请求 Monitor 时，若该 Monitor 已经被其他线程持有，则 t 进入 entry set 等待；
2. 当持有线程释放 Monitor 后，t 需要与 entry set 中的其他等待线程 **一起竞争** 获取 Monitor；
3. 只有其中一个线程会成功，其余竞争失败的线程在 entry set 继续等待；

### 线程协作：wait set

线程协作借助 wait set 实现：

1. 当前 **持有** Monitor 的线程 t 检测到某条件 c 为假，执行 `Object.wait()`，使 t 自己 **立即**：
   1. 自动 **释放** Monitor
   2. 进入 wait set，挂起自己，让出 CPU
2. 线程 s 获取 Monitor，do some work，使 t 依赖的条件 c 变为真后，执行 `Object.notify()`，使：
   1. s 继续正常执行，直到正常 **释放** Monitor 或执行 `Object.wait()` 挂起自己
   2. s 释放 Monitor 后，wait set 中的线程 **重新竞争** 获取 Monitor；
3. 只有其中一个线程会成功，其余竞争失败的线程在 wait set 继续等待；

在等待-通知机制中，共享资源被 Monitor 保护，以提供互斥访问，线程 t 需要的条件会被线程 s 满足，因此 t 和 s 之间一个需要 *等待* 条件被满足，一个需要 *通知* 条件已经满足。

s 发出的 `notify` 通知就 **一定** 能保证条件已经满足了吗？

当然不是，原因有两点：

1. s 执行 `notify` 后，并不会立即停止执行，并释放 Monitor，而是会继续运行，期间 s 可能继续修改共享状态，从而导致 t 等待的条件不再为真；
2. s 释放 Monitor 后，wait set 中的线程会一起竞争获取 Monitor，竞争成功的线程 x 可能会修改 t 等待的条件，使其不为真；

因此 s 发出的 `notify` 仅仅是一种 **条件可能满足** 的提示，t 线程被唤醒后，需要循环检查条件是否为真，若不为真，需要继续 `wait` 等待。

### entry set、wait set 是否有序？

它们可能实现为 FIFO 或 FILO 的队列，也可能实现为无序或有序的 set，这属于 JVM 实现细节，各个厂商可以自由选择。

### 状态转换图

<img src="/images/java/jvm/monitor-entry-wait-set.png" alt="Monitor 图示" style="width: 450px;"/>

上图将 Monitor 相关线程分为 3 类：

1. entry set：**从未** 获取过 Monitor 的线程，排队竞争 Monitor；
2. owner：同一时刻，只有一个线程持有 Monitor；
3. wait set：**曾经持有** Monitor 的线程，通过 `Object.wait()` 主动进入 wait set；

#### 首次进入

要获取 Monitor，通道只有一条，即 entry set，若线程 t 进入 entry set 后发现：

1. Monitor 当前未被持有；
2. entry set 只有自己一个线程在等待；

则 t 可立即获取 Monitor。

#### 释放 Monitor

若 Monitor 当前被线程 t 持有，根据 t 释放 Monitor 前是否执行 `notify`：

1. 执行过 `notify`，则 entry set 和 wait set 中的线程一起竞争 Monitor；
2. 未执行 `notify`，则只有 entry set 中的线程一起竞争 Monitor；

#### 超时版 `wait(t)`

若线程 t 使用带超时参数的 `wait(t)`，则超过 t 时间后，即使没有线程执行 `notify`，t 也会自动从 wait set 中被环境，从而竞争 Monitor，这里超时相当于隐式的 `notify`。

#### `notify` vs `notifyAll`

`notify` 只会从 wait set **随机** 选取一个线程唤醒；而 `notifyAll` 会唤醒 wait set 中的所有线程，不过最多只有一个能成功获取 Monitor，其余会再次进入 wait set。

## HotSpot 实现

HotSpot 虚拟机通过 [ObjectMonitor](https://hg.openjdk.java.net/jdk/jdk/file/896e80158d35/src/hotspot/share/runtime/objectMonitor.hpp) 实现 monitor，与前面的概念模型相似，`ObjectMonitor` 类包含如下关键字段：

1. _EntryList：对应 entry set
2. _WaitSet：对应 wait set
3. _owner：指向持有 Monitor 线程的指针
4. _recursions：重入次数
5. _count：近似等于 |_EntryList| + |_WaitSet|

除 entry set 和 wait set 外，为支持内置锁的 **重入语义**，添加了 `_recursions` 字段，用于记录线程重入次数：

1. 每次获取同一 Monitor，`_recursions` 加 1；
2. 每次释放同一 Monitor，`_recursions` 减 1；

当 `_recursions` 为 0 时，说明可以释放 Monitor 了，否则是重入场景，继续持有 Monitor。
                                     
## 总结

* 内置锁、监视器锁、监视器、管程、Monitor 是同一回事；
* Monitor 功能：
  * 互斥访问：通过 entry set 实现
  * 线程协作：等待 + 唤醒，通过 wait set 实现
* HotSpot 通过 `ObjectMonitor` 类实现 Monitor 机制，支持重入；

---

参考：

* [Java Language Specification 17.1. Synchronization](https://docs.oracle.com/javase/specs/jls/se10/html/jls-17.html#jls-17.1)
* [维基百科 Monitor (synchronization)](https://en.wikipedia.org/wiki/Monitor_(synchronization))
* [Thread Synchronization（好）](https://www.artima.com/insidejvm/ed2/threadsynch.html)
