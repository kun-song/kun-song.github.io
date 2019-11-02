---
title: synchronized 原理（四）：锁优化
date: 2019-11-02 23:26:07
tags:
  - synchronized
categories: JVM
---

Java 1.6 之前，`synchronized` 等价于重量级锁，是通过 Monitor 机制实现的，对应 HotSpot 中的 `ObjectMonitor` 类，该类基于操作系统提供的 **互斥量** 实现，而互斥量会引发线程在 **内核态** 和 **用户态** 之间的切换，导致重量级锁效率较低。

<!-- more -->

`synchronized` 毕竟是 Java 语言原生的锁机制，Java 1.6 引入多项锁优化技术，用于提升 `synchronized` 的性能，包括：

* 自适应自旋
* 锁消除
* 锁粗化
* 轻量级锁 & 偏向锁

## 自适应自旋

`synchronized` 性能损耗最大的地方在于 **挂起**、**恢复** 线程都要转入 **内核态** 完成，根据经验，一般 **锁定状态只会持续很短时间**，例如：

```Java
public synchronized int getAge() {
    return age;
}
```

同步块非常非常短，如果线程 t 执行 `getAget` 时有线程 s 同时尝试执行 `getAge`，若要挂起、恢复线程 s，有点得不偿失，因为内核态、用户态的切换非常耗时，执行时间远远超过 `getAget` 的方法体。

该场景的优化手段即自旋锁 or 自适应自旋。

### 自旋锁

>Java 1.4 引入自旋锁，默认关闭。

所谓的自旋，即原地空转，最简单的实现为 `while(true)`。

线程 s 尝试执行 `getAge` 时，发现锁已经被持有，此时原本 s 需要被 **挂起**，但也可以让 s 原地空转一会（自旋），过一会再次尝试执行 `getAge`，根据 **锁定状态只会持续很短时间** 的假设，第二次有极大可能成功获取锁。

自旋锁的出发点很好，不过有两个问题：

1. 自旋锁适合多核 CPU，若只有一个核心，则不适合；
2. **自旋时间** 难以确定；

如今大部分服务器核心动辄几百，第一个问题已经不是问题。

**自旋时间** 的设置依旧很难：

* 若 **锁占用时间很短**，则自旋锁效果较好；
* 若 **锁占用时间很长**，则自旋时间可能 **远远超过** 线程挂起、恢复的时间；

因此若自旋次数超过一定 **次数限制** 后，仍然没有成功获得锁，则 **放弃自旋**，回退到挂起、恢复线程的方式，默认次数为 10，可通过 `-XX:PreBlockSpin` 修改。

### 自适应自旋

>Java 1.6 优化了自旋锁，引入自适应自旋，默认开启。

自适应自旋是指 **自旋时间自适应**，不再空转固定时间，而是根据同一锁对象上次自旋结果来 **动态调整** 自旋时间：

* 同一锁对象上，自旋刚刚成功获得过锁，则 JVM 认为本次自旋成功概率较大，进而 **延长** 自旋时间；
* 同一锁对象上，自旋 **很少** 成功获得过锁，则 JVM 可能 **减少** 自旋时间，或者直接 **跳过** 自旋；

随着程序运行时间增长，监控信息会不断完善，JVM 对自旋时间的调整会越来越合理。

## 锁消除

与自旋锁基于“经验假设”不同，锁消除的依据是 **逃逸分析**，即一段代码中，虽然对象分配在 **堆** 上，但这些对象都没有逃逸出去，无法被 **其他线程** 访问，效果等价于 **栈** 上数据，属于 **线程私有** 对象，这些对象作为锁毫无意义。

虽然 Java 源码中有 `synchronized` 加锁要求，但 JIT 实际执行时，通过逃逸分析，发现该锁是 **线程私有** 对象，绝对不存在线程竞争，则可消除这些锁的同步。

例如：

```Java
public String cat(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```

代码编写时直接 or 间接会使用很多 `synchronized` 方法，以上 `append` 使用 `sb` 作为锁对象：

```Java
public synchronized StringBuffer append(String str) {
    toStringCache = null;
    super.append(str);
    return this;
}
```

JIT 执行以上代码时，经过逃逸分析，发现 `sb` 是 `cat` 方法的局部变量，而且没有逃逸到 `cat` 方法之外，其他线程无法访问 `sb`，因此用 `sb` 作为锁对象加锁时，**永远不会发生锁竞争**，可以消除所有同步操作。

## 锁粗化

使用 `synchronized` 时，最佳实践是 **尽量减小同步块**，以减少代码 **串行** 部分的比例，提升并发度。

写代码时应该遵守该实践，但有时会不小心 or 无意中违反：

```Java
private StringBuffer sb = new StringBuffer();

public String cat(String s1, String s2, String s3) {
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```

* 这次 `sb` 变成了共享变量，可能存在多线程竞争，无法执行 **锁消除** 优化；

3 个连续的 `append` 操作会连续加锁、解锁 3 次，JVM 检测到这种对 **同一锁对象** 连续的加锁、解锁操作后，会粗化整个同步块，使其包含这 3 个 `append` 调用，从而只需要执行一次加锁、解锁。

## 轻量级锁 & 偏向锁

见 [synchronized 原理（三）：偏向锁、轻量级锁](http://songkun.me/2019/11/02/2019-11-02-java-jvm-synchronized-impl-3/)。
