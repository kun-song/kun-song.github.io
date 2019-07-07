---
title: Java | 守护线程
date: 2018-11-20 08:52:37
tags:
  - Java
categories: Java
---

## 用户线程 vs 守护线程

Java 线程分为两种：

* 用户线程（user thread）
  * 用户线程的优先级较高，JVM 必须等待所有用户线程结束后才能关闭
* 守护线程（daemon thread）
  * 守护线程优先级较低（其实是 **最低**），当 JVM 发现没有运行中的用户线程时，将停止所有守护线程，然后关闭 JVM

守护线程一般为用户线程提供服务，因此用户线程停止后，守护线程 **失去服务对象**，JVM 将其终止也是合情合理。

<!-- more -->

守护线程中的 **死循环** 是无害的，因为它们不会造成 JVM 无法退出，同时若无用户线程运行，守护线程的代码执行立马被停止（包括 `finally`），因此守护线程中 **不要执行 IO 任务**，否则可能导致资源泄漏等问题。

>并非无用户线程运行时，JVM 一定会马上退出，例如在守护线程中调用 `Thread.join()` 就能阻止 JVM 退出，但这种做法非常蛋疼，也没有意义。

## API

守护线程相关的 API 非常简单：

* 创建 `setDaemon`
* 检测 `isDaemon`

创建守护线程非常简单，`Thread.setDaemon(true)` 皆可：

```Java
Thread t = new Thread(...);
t.setDaemon(true);

t.start();
```

* `setDaemon` 必须在 `start` 之前调用，否则抛出 `IllegalThreadStateException` 异常；

因为 **线程将继承其创建者的守护状态**，且 main 线程为用户线程，因此在 main 线程中创建的所有线程都是用户线程，除非通过 `setDaemon` 显式修改。

可通过 `Thread.isDaemon` 查询线程的守护状态：

```Java
t.isDaemon();
```

## 使用场景

一般用于执行 **辅助型的后台任务**，如：

* 垃圾收集
* 删除缓存中的过期记录
* 执行监控任务（心跳发送等）
* 收集监控信息
* 监听请求

-----

参考：

* [Daemon Threads in Java](https://www.baeldung.com/java-daemon-thread)
* [Daemon thread use case](https://stackoverflow.com/questions/18933195/daemon-thread-use-case)
