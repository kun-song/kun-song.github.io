---
title: synchronized 原理（五）：杂项
date: 2019-11-03 00:54:45
tags:
  - synchronized
categories: JVM
---

总结 `synchronized` 常见疑问。

<!-- more -->

1. `synchronized` 可重入：通过 `ObjectMonitor` 的 `_recursions` 实现可重入；
2. `synchronized` 是非公平锁；
3. `synchronized` 阻塞的线程无法被 **中断** 唤醒；