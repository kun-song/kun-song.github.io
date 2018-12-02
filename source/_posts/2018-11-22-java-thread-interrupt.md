---
title: Java | 终止线程
date: 2018-11-22 00:11:17
tags:
  - Java
  - 线程中断
categories: 技术
---

任何非 toy 项目都会用到多线程，启动线程可以用线程池，那终止线程呢，是用 `Thread.stop` 呢，还是 `Thread.sleep` 呢，还是中断呢？

本文介绍终止线程的各种方式，及其最佳实践。

<!-- more -->

## `Thread` 原生支持（已废弃）

`Thread` 类提供了很多貌似可以用于停止线程的方法，下面一一看来。

### `Thread.stop`

从名字看，`Thread.stop` 就是为终止线程而生的：

```
Forces the thread to stop executing.
```

但目前该方法已经被废弃，原因是：

```
This method is inherently unsafe.  Stopping a thread with Thread.stop causes it to unlock all of the monitors that it has locked (as a natural consequence of the unchecked  <code>ThreadDeath</code> exception propagating up the stack).  If any of the objects previously protected by these monitors were in an inconsistent state, the damaged objects become visible to other threads, potentially resulting in arbitrary behavior.  Many uses of <code>stop</code> should be replaced by code that simply modifies some variable to indicate that the target thread should stop running.  The target thread should check this variable regularly, and return from its run method in an orderly fashion if the variable indicates that it is to stop running.  If the target thread waits for long periods (on a condition variable,
 for example), the <code>interrupt</code> method should be used to interrupt the wait.
```



`Thread.stop`、`Thread.suspend`、`Thread.resume` 等方法都已经被废弃



