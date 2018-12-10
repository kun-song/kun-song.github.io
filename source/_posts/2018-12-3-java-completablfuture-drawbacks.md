---
title: CompletableFuture | commonPool vs 自定义线程池
date: 2018-12-03 21:08:39
tags:
  - Java
  - CompletableFuture
categories: 技术
---

`CompletableFuture` 既可以替代传统的线程池，也能实现异步编程，用起来很爽，但其中的坑也不少。

本文将解释一个使用 `CompletableFuture` 时常见的问题，即：要不要提供自定义的线程池。

<!-- more -->

我们知道，`CompletableFuture` 的 API 一般分为两类：

```Java
CompletableFuture.runAsync(() -> System.out.println("Hello World!"));
CompletableFuture.runAsync(() -> System.out.println("Hello World!"), pool);
```

它们的区别在于是否显式提供线程池，不提供线程池时，默认使用 `ForkJoinPool.commonPool()`。

那 `commonPool` 是一个怎样的线程池呢？即使对它不熟悉，从名字也能猜测一二，它是一个会被很多任务 **共享** 的线程池，比如同一 JVM 上的所有 `CompletableFuture`、并行 `Stream` 都将共享 `commonPool`，除此之外，应用代码也能使用它。

`commonPool` 设计时的目标场景是运行 **非阻塞的 CPU 密集型任务**，为最大化利用 CPU，其线程数默认为 `CPU 数量 - 1`：

```Java
// default 1 less than #cores
if (parallelism < 0 && (parallelism = Runtime.getRuntime().availableProcessors() - 1) <= 0)
    parallelism = 1;
```

每个线程有一个双端队列（deque），用于存储自己负责运行的任务，线程池的空闲线程通过 work-stealing 算法“偷取”其他线程的任务，从而提升 CPU 利用率。

`commonPool` 方便共享（直接调用 `ForkJoinPool.commonPool()`），避免了每个任务创建自己的线程池的开销，减少了资源消耗。

`commonPool` 有很多优点，是好东西，我们应该在 `CompletableFuture` 中使用它！

但问题是，`CompletableFuture` 运行的并非一定是 CPU 密集型任务啊，恰好相反，`CompletableFuture` 的一大使用场景为 **异步编程**，异步编程不可避免会涉及很多阻塞操作，而阻塞操作是 `ForkJoinPool` 极力避免的。

详细而言，在 `CompletableFuture` 中使用 `commonPool` 会有如下问题：

首先，`commonPool` 默认线程数较少（CPU总数 - 1），如果在 `CompletableFuture` 中使用，会无意中造成 `commonPool` 运行过多任务，若当前运行的全部是阻塞任务，则其余任务将排在工作线程 deque 中，导致它们无法立即执行。

其次，线程池使用原则之一是：

>不要在同一线程池混合运行阻塞任务和非阻塞任务。

实际使用 `CompletableFuture` 时，一般会混合执行阻塞任务和非阻塞任务，如果使用 `commonPool` 将导致其中部分工作线程一致阻塞，从而降低资源利用率。

最后，`commonPool` 是为 CPU 密集型任务设计的，对于大部分实际应用而言，应该配置比 CPU 数量更多的线程，否则无法充分利用资源。

因此使用 `CompletableFuture` 时，最好要提供自定义的线程池，不要使用默认的 `commonPool`。

---

参考：

* [Which thread executes CompletableFuture's tasks and callbacks?](https://www.nurkiewicz.com/2015/11/which-thread-executes.html)
* [Guide to the Fork/Join Framework in Java](https://www.baeldung.com/java-fork-join)
