---
title: wait() 与 sleep() 的异同点
date: 2019-07-30 22:01:39
tags:
  - Object.wait
  - Thread.sleep
categories: Java
---

本文列举 `wait()` 和 `sleep()` 方法的异同点，力求完整。

<!-- more -->

## 相同点

它们都可以让线程“休眠”一段时间。

在操作系统中，线程有五种状态：

1. 初始状态
2. 可运行状态
3. 运行状态
4. 休眠状态
5. 终止状态

此处的“休眠”一段时间即让线程进入 **休眠状态**。

## 不同点

### JVM 线程状态

前面说过 `wait()` 和 `sleep()` 都可以使线程进入休眠状态，在 JVM 中，休眠状态细分为三种：

1. BLOCKED
2. WAITING
3. TIMED_WAITING

`wait` 根据是否带超时参数，分别进入 WAITING 或 TIMED_WAITING 状态，而 `sleep` 一定进入 `TIMED_WAITING` 状态。

### 释放监视器锁

`wait()` 将释放监视器锁（`synchronized`），而 `sleep()` 不释放锁：

```Java
synchronized(LOCK) {
    LOCK.wait();  // 监视器锁 LOCK 被释放
}

synchronized(LOCK) {
    Thread.sleep(1000);  // 监视器锁 LOCK 保持持有
}
```

>实际上，`X.wait()` 释放的是监视器锁 `X`，其他任何锁都不会释放！

`wait()` 从阻塞状态恢复后，自动尝试 **重新获取** 监视器锁。

### 使用方式

`wait()` 必须在 `synchronized` 块中调用，且使用的监视器锁必须是 `wait()` 的调用对象：

```Java
synchronized(LOCK) {
    LOCK.wait();  // OK：监视器锁为 LOCK，与 wait() 调用对象保持一致
}

synchronized(ANOTHER-LOCK) {
    LOCK.wait();  // ERROR：监视器锁为 ANOTHER-LOCK，与 wait() 调用对象不一致
}
```

若两者不一致，将抛出 `java.lang.IllegalMonitorStateException`。

`sleep()` 使用无任何限制。

### 目标场景

`wait()` 用于线程同步，与 `synchronized`、`notify`、`notifyAll` 一起实现 **等待--通知** 机制（简易版 MESA 模型），用于线程 `t1` 暂停等待：

1. 其他线程 `t2` 完成某项任务；
2. 其他线程 `t2` 工作，以满足某条件；

然后 `t2` 通知 `t1` 任务完成 or 条件满足，使其从休眠状态恢复。

`sleep()` 用于暂停执行，可以响应中断。

中断用于其他线程 `t2` 通知 `t1` 停止工作，终止线程，不过虽然 `sleep` 可以响应中断，但比较少用于终止线程的场景。

### 唤醒条件

`sleep()` 时间到了以后，线程被自动唤醒：

```Java
while (true) {
    System.out.println(LocalDateTime.now());
    Thread.currentThread().sleep(5000);
}

// 2019-07-30T23:20:41.586
// 2019-07-30T23:20:46.590
// 2019-07-30T23:20:51.593
// 2019-07-30T23:20:56.598
...
```

若持有线程的引用，也可通过 **中断** 被唤醒：

```Java
try {
    Thread.currentThread().sleep(5000);
} catch (InterruptedException e) {
    e.printStackTrace();
}

t1.interupt();  // 必须通过直接持有 t1 引用才能唤醒 t1
```

因 `wait()` 休眠的线程无法自动唤醒：

```Java
final Object lock = new Object();
while (true) {
    System.out.println(LocalDateTime.now());
    synchronized (lock) {
        lock.wait();  // lock 对象
    }
}

// 2019-07-30T23:18:54.365
```

必须用 `notify` 或 `notifyAll` 显式唤醒（通常是其他线程）：

```Java
synchronized (lock) {
    lock.notify();  // lock 对象
}
```

>`wait()` 无法被中断唤醒。

注意：

1. `wait`/`notify`/`notifyAll` 都必须在 `synchronized` 块中调用；
2. 必须使用同一个 `lock` 监视器对象，否则无法正确实现等待--通知；
3. 唤醒 `sleep` 线程必须持有线程引用，耦合性高，唤醒 `wait` 线程必须持有监视器对象引用，因为存在某种等待队列，`wait` 和 `notify` 直接交互的是队列，而该队列属于监视器对象；

### 诡异唤醒

`wait()` 除被 `notify`/`notifyAll` 唤醒外，还可能被莫名其妙地唤醒，因此条件检查必须放到 **循环** 中：

```Java
synchronized (lock) {
    while (!condition) {  // 从 wait() 休眠状态醒来后，再次检查条件
      lock.wait();
    }
}
```

### 所在类

`wait()` 定义在 `Object` 类中，而 `sleep()` 定义在 `Thread` 中。

---

参考：

1. [Difference between wait() and sleep()](https://stackoverflow.com/questions/1036754/difference-between-wait-and-sleep)