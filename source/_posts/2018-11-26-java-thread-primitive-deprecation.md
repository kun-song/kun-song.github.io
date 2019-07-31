---
title: 译 | Java Thread Primitive Deprecation
date: 2018-11-26 22:59:23
tags:
  - Thread.stop
  - Thread.resume
categories: Java
---

[原文](https://docs.oracle.com/javase/8/docs/technotes/guides/concurrency/threadPrimitiveDeprecation.html)

## 为何废弃 `Thread.stop` ？

因为它本质上是不安全的。

`stop` 线程将导致释放其持有的全部 monitor（`ThreadDeath` 异常在栈中传播时，monitor 被解锁），若在当前线程中，这些 monitor 保护的对象处于 **不一致状态**，则 `stop` 后这种不一致状态对其他线程可见。

我们称处于不一致状态的对象被“损坏”，当线程操作“损坏”对象时，可能发生任意（无法预测）行为，它们可能非常微妙且难以检测，与其他非检查异常不同，`ThreadDeath` 悄悄杀死线程，用户无法收到任何警告，用户可能到几个小时，或几天会才能发现问题。

<!-- more -->

## 为何不捕获 `ThreadDeath` 并修复“损坏”对象？

理论上可行，实际难度极大，由于两个原因几乎不可能做到：

* 线程几乎可以在 **任何地方** 抛出 `ThreadDeath`，因此所有同步方法、同步代码块都需要仔细设计；
* 处理已捕获的 `ThreadDeath` 时，可能再次抛出 `ThreadDeath`，因此必须不断重复清理，直至成功，代码会非常复杂；

因此，这样做不切实际。

## `Thread.stop(Throwable)` 呢？

虽然有种种问题，`stop(Throwable)` 可以用来抛出目标线程未准备好处理的异常，下面的方法在行为上与 `throw` 子句相同，但不强制方法声明所有受检查异常（`throw` 需要声明）：

```Java
static void sneakyThrow(Throwable t) {
    Thread.currentThread().stop(t);
}
```

>同样被废弃，原因与 `stop()` 相同，不安全。

## 用什么替代 `Thread.stop` ？

大部分 `stop` 使用场景都可以用一个 `volatile` 变量替代，该变量用来指示当前线程是否应该停止运行，其他线程通过修改 `volatile` 变量达到停止线程的目的。

例如：

```Java
private Thread blinker;

public void start() {
    blinker = new Thread(this);
    blinker.start();
}

public void stop() {
    blinker.stop();  // UNSAFE!
}

public void run() {
    while (true) {
        try {
            Thread.sleep(interval);
        } catch (InterruptedException e){
        }
        repaint();
    }
}
```

可修改为：

```Java
private volatile Thread blinker;

public void stop() {
    blinker = null;
}

public void run() {
    Thread thisThread = Thread.currentThread();
    while (blinker == thisThread) {  // 运行标识
        try {
            Thread.sleep(interval);
        } catch (InterruptedException e){
        }
        repaint();
    }
}
```

目标线程检测到停止请求后，可以做必要的资源清理，相比直接释放锁，进而处于不一致状态要好很多。

## 如何停止长时间等待的线程（如等待输入）？

`Thread.interrupt` 正是为此设计的。

上面“基于状态”的信号机制也可以使用，但状态修改后，可以跟一个 `Thread.interrupt` 调用，从而 **中断等待**：

```Java
public void stop() {
    Thread moribund = waiter;
    waiter = null;
    moribund.interrupt();
}
```

>* `waiter == null` 可在下次 `while(waiter == thisThread)` 中检测到退出信号，若线程等待，则 `while()` 将无法立即退出；
>* `moribund.interrrupt()` 保证等待线程能立即退出；

为保证中断正常工作，任何捕获中断异常，且无法合理处置的方法都 **必须立即重新中断**：

```Java
Thread.currentThread().interrupt();
```

此处说重新中断，而非重新抛出中断异常，是因为若方法未声明抛出 `InterruptedException` 就无法抛出，此时只能 `interrupt`。

## 线程未响应 `Thread.interrupt` 怎么办？

部分场景，可在应用中使用一些特殊技巧，例如若线程在已知 socket 等待，可关闭 socket，线程将立即返回。

但不幸的是，没有通用技巧，且任何不响应 `Thread.interrupt` 的线程，也不会响应 `Thread.stop`，例如 DDOS 攻击等。

## `Thread.suspend` 和 `Thread.resume` 也被废弃了？

`Thread.suspend` 容易导致死锁。

若线程一 `suspend` 时通过 monitor A 保护稀有资源，则 `suspend` 后线程一 **不释放 minotor A**，因此其他线程无法访问该资源，直至线程一 `resume`。若恢复线程一的线程在调用 `resume` 前需要获取 monitor A，则发生死锁。

>`resume` 为服务 `suspend` 而存在。

对此 `Thread.suspend` 方法的注释也有解释：

```
This method has been deprecated, as it is inherently deadlock-prone.

If the target thread holds a lock on the monitor protecting a 
critical system resource when it is suspended, no thread can 
access this resource until the target thread is resumed. 

If the thread that would resume the target thread attempts 
to lock this monitor prior to calling resume, deadlock results.  

Such deadlocks typically manifest themselves as "frozen" processes.
```

`suspend` 容易死锁的根因是它 **不释放锁**，`resume` 它的线程如果要请求同样的锁，则挂起线程永远无法恢复。

## 用什么替代 `Thread.suspend` 和 `Thread.resume` ？

与 `Tread.stop` 类似，明智的办法是添加一个 `volatile` 状态变量（active or suspended），目标线程不断测试该变量，若目标状态是 suspended，则目标线程通过 `Object.wait` 阻塞，resume 时，通过 `Object.notify` 唤醒目标线程。

例如：

```Java
private boolean threadSuspended;

Public void mousePressed(MouseEvent e) {
    e.consume();

    if (threadSuspended)
        blinker.resume();
    else
        blinker.suspend();  // DEADLOCK-PRONE!

    threadSuspended = !threadSuspended;
}
```

可修改为：

```Java
public synchronized void mousePressed(MouseEvent e) {
    e.consume();

    threadSuspended = !threadSuspended;

    if (!threadSuspended)
        notify();
}
```

然后在 `run` 循环中添加：

```Java
synchronized(this) {
    while (threadSuspended)
       wait();
}
```

因为 `wait` 会抛出 `InterrupedException`，因此 `wait` 必须放在 `try catch` 中，最后 `run` 如下：

```Java
public void run() {
    while (true) {
        try {
            Thread.sleep(interval);

            synchronized(this) {
                while (threadSuspended)
                    wait();
            }
        } catch (InterruptedException e){
        }
        repaint();
    }
}
```

* `sleep` 放在 `wait` 之前，可以保证线程恢复后立马执行 `repaint`。

注意 `notify` 和 `wait` 都在 `synchronized` 代码块中，这是为了消除竞态条件，避免挂起线程错过 `notify` 消息，进而永远挂起。

虽然随着 JVM 的成熟，`synchronized` 代价越来越低，但总归有代价。可借助一点小技巧避免每次循环时都要加锁：

```Java
if (threadSuspended) {
    synchronized(this) {
        while (threadSuspended)
            wait();
    }
}
```

* 以前每次循环都加锁，现在只有挂起时（即 `threadSuspended` 为 `true`）时才加锁；

现在 `threadSuspended` 没有 `synchronized` 保护了，需要声明为 `volatile`。

最后 `run` 为：

```Java
private volatile boolean threadSuspended;

public void run() {
    while (true) {
        try {
            Thread.sleep(interval);

            if (threadSuspended) {
                synchronized(this) {
                    while (threadSuspended)
                        wait();
                }
            }
        } catch (InterruptedException e) {
        }
        repaint();
    }
}
```

## 能让线程既可安全 stop，也能安全 suspend 吗？

当然可以，唯一要注意的是，停止目标线程时，目标线程可能已经被挂起。若 `stop` 仅仅将状态变量（`blinker`）设为 `null`，则目标线程仍会保持挂起，而不是优雅退出。

为修正该场景，`stop` 须保证目标线程处于挂起状态时能立即恢复（resume），一旦恢复，目标线程必须立即感知自己已经被停止，然后优雅停机。

如：

```Java
public void run() {
    Thread thisThread = Thread.currentThread();
    while (blinker == thisThread) {
        try {
            Thread.sleep(interval);

            synchronized(this) {
                while (threadSuspended && blinker == thisThread)
                    wait();
            }
        } catch (InterruptedException e) {
        }
        repaint();
    }
}

public synchronized void stop() {
    blinker = null;
    notify();
}
```

若 `stop` 调用 `Thread.interrupt`，就不用调用 `notify`，但 `stop` 还是必须用 `synchronized` 修饰，同步可以保证目标线程不会因为竞态条件而错误中断。

## `Thread.destroy` 呢？

`Thread.destroy` 从来没被实现，并且已被废弃。

即使实现了 `destory`，与 `Thread.suspend` 类似，`destroy` 容易导致死锁。
