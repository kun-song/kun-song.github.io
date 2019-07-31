---
title: 如何安全地终止线程？
date: 2019-07-27 23:18:31
tags:
  - 中断
  - Thread.stop
categories: Java
---

在多线程编程中，有多种方式可以终止线程，但要“安全地”终止线程却没那么容易。

Java **没有提供** 可以直接安全终止线程机制（`Thread.stop` 虽可直接终止线程，但并不安全），这是合理的，因为通常 **只有线程自己** 才知道该如何安全终止自己（资源释放、状态回滚等），因此 Java 使用协作的终止方式，即控制线程发出终止目标线程的请求，目标线程检测到该请求后，首先执行必要的 **清理工作**，然后安全地终止自己。

<!-- more -->

## 被废弃的 `Thread.stop`

终止线程最直观的方式是 `Thread.stop`，不过该方法早就被废弃了，看下注释：

>This method is inherently unsafe. 
>
>Stopping a thread with Thread.stop causes it to **unlock all of the monitors** that it has locked (as a natural consequence of the unchecked **ThreadDeath**s exception propagating up the stack).  If any of the objects previously protected by these monitors were in an **inconsistent state**, the damaged objects become **visible** to other threads, potentially resulting in arbitrary behavior.  

`Thread.stop` 会粗暴地立刻停止，有两个后果：

1. **立即释放** 线程持有的监视器锁；
2. **无法自动调用** `Lock.unlock()`，从而无法释放线程持有的 `Lock` 锁；

立即释放监视器锁后，以前被这些锁保护的 **共享状态** 立刻 **对其他线程可见**，因为释放锁的那一刻线程执行情况未知，所以共享状态可能 **违反一致性协议**，最终结果无法预料。

而不释放 `Lock` 锁，则其他线程再也无法获取该锁，应用肯定不正常。

例如下面的同步方法修改了 `target` 和 `max` 两个状态变量：

```Java
private synchronized freeLockAbruptly() {
    target = findTarget();
    if (target > max) {
        max = target;
    }
}
```

如果 `findTarget()` 返回后线程被 `stop()` 强制终止，且 `max` 应该更新为 `findTarget()` 的结果，则其他线程可以看到 `target` 已经更新，但 `max` 却没及时更新，导致程序状态不一致。

除可能产生不一致的共享状态外，`Thread.stop()` 无法让线程执行 **资源释放** 等操作，可能造成资源泄漏。

注释也提到建议的替代方案：

>Many uses of stop should be replaced by code that simply modifies some **variable** to indicate that the target thread should stop running.  The target thread should **check this variable regularly**, and return from its run method in an orderly fashion if the variable indicates that it is to stop running.
>
>If the target thread **waits for long periods** (on a condition variable, for example), the interrupt method should be used to interrupt the wait.

普通场景用 **状态变量 + 轮询检查** 即可，若线程中存在 **阻塞方法** 或 **耗时操作**，则可用 **轮询 + 中断**。

由于该状态标志必须允许多线程同时更新、读取，为保证 **可见性**，必须用 `volatile` 修饰，这也是 `volatile` 最典型的应用场景之一。

## `volatile` 状态变量 + 轮询

该方式有两个关键点：

1. 用 `volatile` 状态变量表示其他线程的终止请求；
2. 线程内部不断轮询判断该标志，若被设置，则终止线程；

例如：

```Java
public class PrimeGenerator implements Runnable {
    private final List<BigInteger> primes = new ArrayList<>();

    private volatile boolean cancelled;  // 共享标志位，volatile 保证可见性

    @Override
    public void run() {
        BigInteger p = BigInteger.ONE;
        while (!cancelled) {  // 循环检查
            p = p.nextProbablePrime();
            synchronized (this) {
                primes.add(p);
            }
        }
    }

    public void cancel() { cancelled = true; }

    public synchronized List<BigInteger> getPrimes() {
        return new ArrayList<>(primes);  // 保护复制
    }
}
```

假设想要获取 1s 内产生的素数集合：

```Java
public List<BigInteger> getPrimesOfOneSecond() throws InterruptedException {
    PrimeGenerator generator = new PrimeGenerator();
    new Thread(generator).start();  // 线程 1

    try {
        TimeUnit.SECONDS.sleep(1);
    } finally {
        generator.cancel();  // 线程 2
    }

    return generator.getPrimes();
}
```

>注意：为保证线程 2 被中断时也能终止线程 1，需要把 `generator.cancel()` 放到 `finally` 块中执行；

通过在线程 2 中执行 `generator.cancel()` 将线程 1 中的状态变量 `cancelled` 设置为 `true`，从而取消了线程 1 正在执行的任务，最后线程 1 终止执行。

该方式存在一定缺陷：线程 1 只能在执行完本次循环体之后才能检测到终止状态，因此若循环体内：

1. 存在 **耗时** 方法，则终止动作需要长时间才能被影响；
2. 存在 **阻塞** 方法，则永远无法终止；

## 轮询 + 中断

轮询中存在耗时方法，没有特别好的解决方式，但对于阻塞方法，则有标准的解决方式，即中断。

>在 Java 中，中断就是为线程协作而生，用于实现任务取消、线程终止等。

每个线程都有一个 `boolean` 类型的 **中断状态**，中断操作只是设置了目标线程的中断状态，目标线程检测到中断后可以响应，也可以什么都不做，中断本身无法直接终止正在执行的线程，仅仅是一种 **通知机制**。

Java 中的大部分阻塞方法都支持中断，因此终止线程最标准的做法就是中断。

>阻塞方法支持中断的含义：处于阻塞状态时可检测到中断请求，抛出 `InterruptedException`，并恢复本线程的中断状态。

使用线程本身的中断状态取代 `cancelled` 标识：

```Java
public class PrimeGenerator implements Runnable {
    private final BlockingQueue<BigInteger> primes = new ArrayBlockingQueue<>(10);

    @Override
    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted()) {  // 检查中断
                p = p.nextProbablePrime();
                primes.put(p);  // 阻塞方法
            }
        } catch (InterruptedException consumed) {
            // put 方法检测到中断后，抛出异常，终止线程
        }
    }

    public void cancel() { cancelled = true; }

    public synchronized List<BigInteger> getPrimes() {
        return new ArrayList<>(primes);  // 保护复制
    }
}
```

使用如下：

```Java
public List<BigInteger> getPrimesOfOneSecond() throws InterruptedException {
    PrimeGenerator generator = new PrimeGenerator();
    Thread t = new Thread(generator);
    t.start();

    try {
        TimeUnit.SECONDS.sleep(1);
    } finally {  // 在 finally 块中执行中断
        t.interrupt();
    }

    return generator.getPrimes();
}
```

对于该示例，只捕获中断异常然后终止线程也可以，这里多走了一步：不断循环检查 `!Thread.currentThread().isInterrupted()`，这是为了尽量提升响应性，因为 `put` 调用结束后、`while` 下次检查前之间也可能被中断，若无该循环检查，则线程只能到下次调用 `put` 时才能终止。

注意：本例捕获了 `InterruptedException` 但 `catch` 中什么都没做，因为本例捕获到中断后想要终止线程，什么都不做正好达成了目标，因此吞掉中断异常，进而终止线程就是我们的 **中断策略**。

如果没有清晰的中断策略，千万不要 **直接吞掉** 中断，有两种常见处理方式：

1. 将 `InterruptedException` 向上抛出；
2. 通过 `Thread.currentThread().interrupt()` 恢复本线程的中断状态；
