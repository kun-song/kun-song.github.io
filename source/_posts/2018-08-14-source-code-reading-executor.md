---
title: 源码夜读 | Executor
date: 2018-08-14 12:33:37
tags:
  - Java
  - Executor
categories: 源码夜读
---

在 Java 并发编程中，`Runnable` 是逻辑上的工作单元，用来表示任务，而 `Thread` 是执行任务的机制，不过 `Thread` 过于底层、过于基础，因此需要复杂的资源管理，比如创建多少线程合适？当线程崩溃会如何处理？如何停止进行中的任务？

因此一般不直接使用 `Thread` 执行任务，取而代之的是 `Executor` 接口：

```Java
public interface Executor {
    void execute(Runnable command);
}
```

<!-- more -->

虽然 `Executor` 接口非常简单，但它明确解耦了

1. 任务提交
2. 任务的实际 **运行机制**（线程创建、使用、调度等）

任务提交通过 `execute` 方法完成，而任务的执行机制则由各 `Executor` 实现负责，通过替换 `Executor` 实现，可轻易在不同运行机制之间切换。

## 最简单 `Executor` 实现

`Executor` 并未强制要求任务 **异步** 执行，因此最简单的实现是直接运行 `Runnable`：

```Java
public class DirectExecutor implements Executor {
    @Override
    public void execute(Runnable command) {
        command.run();
    }
}
```

此时任务在 **调用者线程** 中运行：

```Java
public static void main(String[] args) {
    Executor executor = new DirectExecutor();

    executor.execute(() -> System.out.println(Thread.currentThread()));
    executor.execute(() -> System.out.println(Thread.currentThread()));
}

// 输出
Thread[main,5,main]
Thread[main,5,main]
```

## 多线程 `Executor`

一般而言，`Executor` 会在独立线程中执行任务：

```Java
public class ThreadPerTaskExecutor implements Executor {
    @Override
    public void execute(Runnable command) {
        new Thread(command).start();
    }
}
```

此时，每个任务运行在独立线程中：

```Java
public static void main(String[] args) {
    Executor executor = new ThreadPerTaskExecutor();

    executor.execute(() -> System.out.println(Thread.currentThread()));
    executor.execute(() -> System.out.println(Thread.currentThread()));
}

// 输出
Thread[Thread-0,5,main]
Thread[Thread-1,5,main]
```

## 串行 `Executor`

很多 `Executor` 实现会对任务的执行、调度添加很多约束，例如：

```Java
public class SerialExecutor implements Executor {
    private final Queue<Runnable> tasks = new ArrayDeque<>();
    private final Executor executor;
    private Runnable active;

    public SerialExecutor(Executor executor) {
        this.executor = executor;
    }

    @Override
    public synchronized void execute(Runnable command) {
        tasks.offer(() -> {
            try {
                command.run();
            } finally {
                scheduleNext();
            }
        });

        if (active == null) {
            scheduleNext();
        }
    }

    private synchronized void scheduleNext() {
        if ((active = tasks.poll()) != null) {
            executor.execute(active);
        }
    }
}
```

* `execute` 和 `schduleNext` 中有很多 **复合操作**，通过同步实现其原子性；

使用：

```Java
public static void main(String[] args) {
    Executor fixedThreadExecutor = Executors.newFixedThreadPool(10);
    Executor executor = new SerialExecutor(fixedThreadExecutor);

    executor.execute(() -> System.out.println(Thread.currentThread()));
    executor.execute(() -> System.out.println(Thread.currentThread()));
}

// 输出
Thread[pool-1-thread-1,5,main]
Thread[pool-1-thread-2,5,main]
```

## 标准库 `Executor` 实现

`java.util.concurrent` 包中的 `Executor` 实现还额外实现了 `ExecutorService` 接口，以获取对 executor 生命周期的管理，常见的 `Executor` 实现有：

