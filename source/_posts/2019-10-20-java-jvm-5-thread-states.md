---
title: Java 线程的生命周期
date: 2019-10-20 09:45:50
tags:
  - 线程状态
categories: JVM
---

Java 线程模型基于底层操作系统的 **线程模型** 实现，每个 Java 线程都与一个 OS 线程 **一一对应**。

OS 和 JVM 都有线程生命周期的概念，它由一系列 **线程状态** 构成，描述了一个线程生老病死的状态变化，但两者的线程状态并非一一对应。

<!-- more -->

## OS 线程状态

OS 定义的线程生命周期一般由 5 个状态组成：

<img src="/images/java/os-thread-state.png" alt="OS 线程状态转换图" style="width: 600px;"/>

1. 初始状态
   1. 线程在 **编程语言层面** 被创建，只是内存中的一段数据，OS 无法感知它的存在，在 OS 看来，线程还 **未被创建**；
   2. 不允许分配 CPU 执行（因为 OS 线程还没被创建，想分配 CPU 也不可能）；
2. 可运行状态
   1. 对应的 OS 线程已经被创建成功；
   2. 可分配 CPU 执行；
3. 运行状态
   1. 当有 **空闲 CPU** 时，OS 可将其分配给一个处于 **可运行状态** 的线程，该线程进入运行状态；
   2. 占用 CPU；
4. 休眠状态
   1. 处于运行状态的线程若调用阻塞 API or 等待某事件（如条件变量）就会进入休眠状态；
   2. 释放 CPU；
   3. 阻塞 API 返回 or 等待的时间发生后，OS 重新调度，可再次获得 CPU；
5. 终止状态
   1. 线程正常 or 异常执行结束；


不同编程语言会对这 5 种状态进行合并 or 细化：

* C 语言把初始状态、可运行状态合并为一个状态；
* Java 语言
  * 把 **可运行状态** 和 **运行状态** 合并为可运行状态；
  * 把 **休眠状态** 细化为 **阻塞状态**、**限时等待** 和 **不限时等待** 3 个状态；

## Java 线程状态

### 基本概念

Java 共有 6 种线程状态：

<img src="/images/java/java-thread-state.png" alt="Java 线程状态转换图" style="width: 600px;"/>

1. NEW（初始状态）
2. RUNNABLE（可运行状态 + 运行状态）
3. BLOCKED（阻塞状态，属于休眠状态）
4. WAITING（无限时等待，属于休眠状态）
5. TIMED_WAITING（限时等待，属于休眠状态）
6. TERMINATED（终止状态）

其中：

* NEW 和 TERMINATED 对应 OS 的初始状态、终止状态；
* RUNNABLE 对应 OS 的可运行状态 + 运行状态，因为这两个状态对 OS 进行 **线程调度** 有用，但 JVM 依赖 OS 调度，本身不关心调度细节，自然无需区分；
* BLOCKED + WAITING + TIMED_WAITING 都对应 OS 的休眠状态，可视为 **导致休眠状态的三大原因**；

### 状态转换

在 OS 线程模型中，最重要的状态转换关系是可运行状态、运行状态、休眠状态 3 者之间的相互转换，它们涉及 OS 线程调度的知识；在 JVM 中，对应的是 RUNNABLE、BLOCKED、WAITING、TIMED_WAITING 4 者之间的相互转换，更加细致。

#### 1. RUNNABLE <-> BLOCKED

在 OS 层面对应 **运行状态 <-> 休眠状态**，虽然 JVM 的 RUNNABLE 包含可运行状态 + 运行状态，但 OS 不允许 **可运行状态 <-> 休眠状态**，JVM 也要遵循客观规律。

只有一种场景会触发该转换，即线程等待 or 成功获取 `synchronized` 使用的监视器锁（同步块、同步方法）：

* RUNNABLE -> BLOCKED：线程等待监视器锁；
* BLOCKED -> RUNNABLE：线程成功获取监视器锁；

>注意：调用阻塞 API 不会导致线程进入 JVM 层面的 BLOCKED 状态，但会导致线程进入 OS 层面的休眠状态。

#### 2. RUNNABLE <-> WAITING

有 3 个方法会触发该转换：

1. `Object.wait()` 和 `Object.notify()/notifyAll()`
2. `Thread.join()`
3. `LockSupport.park()` 和 `LockSupport.unpark()`

###### `wait()` 和 `notify()/notifyAll()`

`wait` + `notify/notifyAll` 可构建 **条件队列**，它们都必须在 `synchronized` 块 or 方法中使用。

假设有线程 A B 分别调用 `wait` 和 `notify/notifyAll`，转换路径为：

* RUNNABLE -> WAITING：A 先获得 `synchronized` 监视器锁，然后调用 `Object.wait()`，则
  * 线程 A 进入 WAITING 状态；
  * 线程 A 释放前面获取到的 `synchronized` 监视器锁；
* WAITING -> RUNNABLE：B 先获得 `synchronized` 监视器锁，然后调用 `Object.notify/notifyAll`，则
  * 线程 A 被唤醒，进入 RUNNABLE 状态；

>更多关于 `wait` 和 `notify/notifyAll` 可参考 [wait() 与 sleep() 的异同点](http://songkun.me/2019/07/30/2019-07-30-java-difference-between-wait-and-sleep/)。

###### `Thread.join()`

假设有线程 A B，并且在 A 中执行 `B.join()`，转换路径为：

* RUNNABLE -> WAITING：A 执行完 `B.join()` 后进入 WAITING 状态；
* WAITING -> RUNNABLE：B 执行结束后，A 恢复 RUNNABLE 状态，继续执行；

###### `park()` 和 `unpark(thread)`

`LockSupport.park/unpark` 被设计取代 `Thread.suspend/resume`，用于挂起、恢复线程（[谈谈 LockSupport 对 suspend/resume 的改进](http://songkun.me/2019/07/12/2019-07-12-java-locksupport-vs-suspend-resume/)）。

假设有线程 A B，转换路径为：

* RUNNABLE -> WAITING：B 执行 `LockSupport.park()` 挂起自己，让自己进入 WAITING 状态；
* WAITING -> RUNNABLE：A 执行 `LockSupport.unpark(B)` 恢复 B，让 B 进入 RUNNABLE 状态；

#### 3. RUNNABLE <-> TIMED_WAITING

有 5 个方法会触发该转换：

1. `Thread.sleep(millis)`
2. `Object.wait(millis)` 和 `Object.notify/notifyAll`
3. `Thread.join(millis)`
4. `LockSupport.parkNanos(Object blocker, long deadline)`
5. `LockSupport.parkUntil(long deadline)`

其中后面 4 个分别是 `wait()`、`join()`、`park()` 等 **带超时时间的版本**，使用原来与前面相同。

`Thread.sleep(millis)` 必须携带超时参数，转换路径为：

* RUNNABLE -> TIMED_WAITING：线程 A 执行 `Thread.sleep(millis)` 休眠自己，让自己进入 TIMED_WAITING 状态；
* TIMED_WAITING -> RUNNABLE：休眠时间结束后，A 再次进入 RUNNABLE；

#### 4. NEW -> RUNNABLE

首先新创建的线程处于 NEW 状态，NEW 状态的线程仅存在于 Java 语言层面，OS 无法感知，也 **无法调度执行**：

```Java
// t 处于 NEW 状态
Thread t = new Thread(() -> System.out.println("hello world"));
```

想要 `t` 被 OS 调度执行，必须将它从 NEW 转换为 RUNNABLE，有且仅有调用 `Thread.start()` 一种方式：

```Java
// NEW -> RUNNABLE
t.start();
```

前面说过 JVM 层面的 RUNNABLE 状态对应 OS 的：

* 可运行状态
* 运行状态

因此执行完 `t.start()` 后并不意味着线程立即开始执行了，只是说 JVM 层面认为线程已经开始执行了，实际上需要 OS 真正调度分配 CPU 后才算执行，不过 JVM 不关心调度而已。

#### 5. 其他状态 -> TERMINATED 

从除 TERMINATED/NEW 以外的 4 种状态都可以转换到 TERMINATED：

1. 线程执行完 `run()` 方法（正常执行完 + 抛出异常），会自动进入 TERMINATED 状态；
2. 提前终止线程

关于第二点，参考 [如何安全地终止线程？](2019-07-27-java-how-to-stop-a-thread-correctly.md)，有两种终止方式：

1. `volatile` 状态变量 + 轮询
   1. RUNNABLE -> TERMINATED
2. 中断 + 轮询
   1. RUNNABLE -> TERMINATED
   2. BLOCKED -> RUNNABLE -> TERMINATED
   3. WAITING -> RUNNABLE -> TERMINATED
   4. TIMED_WAITING -> RUNNABLE -> TERMINATED

##### RUNNABLE -> TERMINATED

可以看到，转换路径其实只有一条，即 RUNNABLE -> TERMINATED，要终止 **休眠状态**（BLOCKED/WAITING/TIMED_WAITING）的线程，需要先将其从休眠状态唤醒到 RUNNABLE，然后再终止。

`volatile` 状态变量 or 中断（2.1）都可以使线程从 RUNNABLE -> TERMINATED，它们的用法、原理都是类似的，唯一区别是表示是否终止的状态变量是 `volatile` 标志 or 线程本身的中断标志而已。

##### 休眠状态 -> TERMINATED

中断另外 3 种用法（2.2/3/4）则不同，它们通过 **抛出 `InterruptedException` 异常** 将线程从休眠状态唤醒到 RUNNABLE，然后按照 RUNNABLE -> TERMINATED 路径转换即可。

前面总结过使线程 RUNNABLE -> BLOCKED/WAITING/TIMED_WAITING 的几个方法：

* RUNNABLE -> BLOCKED：`synchronized`
* RUNNABLE -> WAITING：`join()`、`wait/notify`、`park/unpark`
* RUNNABLE -> TIMED_WAITING：`join(t)/sleep(t)`、`wait(t)`、`parkUntil/parkNanos`

这些方法的签名都声明了 `InterruptedException`，即它们都能响应中断（调用 `t.interrupt()`），**即使处于休眠状态**。

##### 特殊：RUNNABLE 阻塞

Java NIO 有两个方法比较特殊，调用时线程处于 JVM 层面的 RUNNABLE 状态，但处于 OS 层面的阻塞状态（？）：

* `java.nio.channels.InterruptibleChannel`
* `java.nio.channels.Selector`

线程调用它们时，不会进入 BLOCKED/WAITING/TIMED_WAITING 等，但它们可能长时间阻塞等待，使用 **轮询** 很难处理。

幸运的是这两个方法都能 **响应中断**，即其他线程执行 `t.interrupt()` 时，若 t 处于 RUNNABLE 且正在阻塞在 `InterruptibleChannel` 或 `Selector` 上，则它们分别如此响应中断：

* `InterruptibleChannel` 抛出 `ClosedByInterruptException`
* `java.nio.channels.Selector` 会立即返回，不抛出异常

不管是抛出异常 or 立即返回，线程都从阻塞状态恢复了，可以继续终止。

