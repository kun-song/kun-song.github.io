---
title: volatile 初探
date: 2018-05-24 21:41:45
tags:
  - volatile
categories: Java
---

并发编程中的 **同步机制** 可以实现：

1. 对共享状态的排他访问
1. 内存可见性

Java 保证 **共享变量** 一致性的方式有：

* `synchronized`（监视器锁）
* `volatile` 变量
* CAS（如 `AtomicInteger`）
* 并发包中的高级抽象

<!-- more -->

## `synchronized` 的劣势

`synchronized` 是最古老的同步机制，通过监视器锁获取对共享变量的 **排他** 访问，肯定能保证一致性性，但有如下缺点：

* 上下文切换开销很大，若使用锁的时间比上下文切换短的多，则得不偿失
* 线程调度
  + 多线程竞争锁时，只有一个线程会成功获取锁，其余线程将 **挂起** 等待
  + 挂起的线程恢复执行时，又必须等待其他线程执行完自己的 **时间片** 后，才能被调度执行
  + 挂起线程、恢复线程开销很大，且中间存在较长时间的中断
  + 线程挂起、等待执行时，不能做任何其他事情

`volatile` 是非常轻量级的 `synchronized`，因为使用 `volatile` 变量时不会发生 **上下文切换** 和 **线程调度**，从内存可见性角度看：

* 写入 `volatile` 变量相当于 **退出** 同步代码块
* 读取 `volatile` 变量相当于 **进入** 同步代码块

>`synchronized` 既可以保证可见性，又可以保证原子性；`volatile` 变量只能保证可见性。

`volatile` 可以保证 **共享变量** 的内存可见性，即多线程访问共享变量时，一个线程修改了该变量，其他线程读取的都是修改后的 **最新值**。

Java 语言规范对 `volatile` 定义如下：

>The Java programming language allows threads to access **shared variables**. As a rule, to ensure that shared variables are **consistently** and **reliably** updated, a thread should ensure that it has exclusive use of such variables by obtaining a lock that, conventionally, enforces mutual exclusion for those shared variables.
>
>The Java programming language provides a second mechanism, volatile fields, that is more convenient than locking for some purposes.
>
>A field may be declared volatile, in which case the Java Memory Model ensures that all threads see a consistent value for the variable.

简单讲，多线程共享可变状态时，为确保共享状态被 **准确**、**一致** 地的更新，线程应该通过 **排他锁** 获取该变量。

若不用锁，也可用 `volatile` 字段，Java 内存模型将保证 **所有线程** 间 `volatile` 变量视图的一致性。

虽然 `volatile` 比 `synchronized` 开销小，但 `volatile` 有自己的局限性：

* `volatile` 可以实现与 `synchronized` 类似的可见性，但无法构建 **原子的复合操作**

因此，若一个变量依赖于其他变量，或变量的新值依赖于旧值时，就无法使用 `volatile`。

## `volatile` 保证可见性

简单回顾下出现内存可见性问题的原因，现代计算机架构中，每个 CPU 有专属缓存，多个 CPU 会 **共享内存**，读写操作经历多个步骤：

1. 读取数据时，数据从内存达到缓存，又从缓存到达 CPU 寄存器
1. 写入数据时，CPU 将数据写入 CPU 专属缓存，再未来某一时刻，将缓存数据刷新的内存

因此多个 CPU 处理同一个变量（共享变量）时：

1. 读取的可能是缓存中的值，导致每个 CPU 读到的值都可能不同
1. 写入时仅写入缓存，导致其他 CPU 短时间根本看不到这次修改的内容

通过 CPU 提供的 `Lock` 指令，`volatile` 修饰的共享变量，在执行 **写操作** 时，会：

1. 将本 CPU 缓存中的数据回写到 **内存**
2. 第 1 条的缓存回写导致，使其他 CPU **缓存** 中的该变量数据失效

结果多线程访问 `volatile` 变量时，各个线程总是能获取到 `volatile` 变量的 **最新值**，从而保证内存可见性。

## `volatile` 不保证原子性

`volatile` 只能保证可见性，不能保证 **原子性**，例如：

```Java
public class Counter {
    private volatile int count;

    public void increment() {
        try {
            TimeUnit.MILLISECONDS.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        count++;  // or count = count + 1
    }

    public int getCount() {
        return count;
    }
}
```

当多线程共享 `Counter` 实例时：

```Java
Counter counter = new Counter();
CountDownLatch latch = new CountDownLatch(10000);

for (int i = 0; i < 10000; i++) {
  Thread t = new Thread(() -> {
    latch.countDown();
    counter.increment();
  });
  t.start();
}

latch.await();

System.out.println(counter.getCount());
```

最后 `Counter.getCount()` 值几乎不可能是 10000，因为 `count = count + 1` 或 `count++` 都不是原子操作，它们实际上由 3 个原子操作组成：

1. 读取 `count` 的值
2. 执行 + 1 运算
3. 将 `count` 的值写回内存

当 `count` 不是 `volatile` 变量，则执行完 `count++` 后，结果写入 CPU 缓存，并在未来某时刻刷新到内存，因此其他线程不保证马上看到 `count++` 后 `count` 的最新值，即无法保证内存可见性。

当 `count` 是 `volatile` 变量时，执行完 `count++`，结果马上写入内存，并使其他 CPU 的缓存失效，其他 CPU 读取 `count` 时只能去内存中读取，因此必然能看到 `count` 的最新值。

但是 `volatile` 无法保证 1、2、3 步整体是原子操作，线程 `A` 执行 1、2、3 时，其他线程也可以执行 1、2、3，因此某一时刻完全可能有多个线程 **同时** 执行步骤 1，然后两个线程计算结果相同：

* 线程 A 读取 `count` 值，获得 100，计算得到 101，将 101 写回内存
* 线程 B 读取 `count` 值，获取 100，计算得到 101，将 101 写回内存

该场景下只能用锁，或 `AtomicInteger` 之类的原子类，而不能用 `volatile`。

## 什么时候使用 `volatile` ？

`volatile` 变量一般用作某操作完成、中断的 **标志**，或者状态的 **标志**。

因为 `volatile` 无法保证原子性，所以使用必须非常消息，仅当满足以下所有条件时，才可以用 `volatile` 变量：

1. 对变量的写入操作不依赖于其当前值，或确保只有 **一个** 线程可以更新变量的值；
1. 该变量不会与其他状态变量一起纳入不变性条件；
1. 访问该变量时不需要加锁；

注意 `volatile` 标识符只允许两个操作：

* 读取
* 单纯的写入

标志符的写操作不能 **依赖** 旧值（想想 `Counter` 的例子），若能保证这点，则使用 `volatile` 非常合适，当然也可以使用 `synchronized`，但锁会带来上下文切换、线程调度等开销，比 `volatile` 重了不少。

---

参考：

* 《Java 并发编程实战》
* 《Java 并发编程的一书》
* [Java 语言规范之 volatile](https://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.3.1.4)
