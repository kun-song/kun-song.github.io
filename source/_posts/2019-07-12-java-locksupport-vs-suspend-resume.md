---
title: 谈谈 LockSupport 对 suspend/resume 的改进
date: 2019-07-12 08:45:20
tags:
  - LockSupport
  - suspend
  - resume
categories: Java
---

`LockSupport` 是 Java 并发包中一个比较底层的类，用于实现 JUC 中的并发工具，是 JUC 的基础。

`LockSupport` 提供的 `park`/`unpark` 方法，是对 `Thread.suspend`/`Thread.resume` 的改进，解决了 `suspend`/`resume` 方法的缺陷，提供更加强大的功能。

<!-- more -->

## `suspend`/`resume` 的缺陷

`Thread` 类的 `suspend`/`resume` 是一对挂起、恢复线程的方法，大多成双成对使用，但它们存在几个缺陷，已经被废弃（[译 | Java Thread Primitive Deprecation](http://songkun.me/2018/11/26/2018-11-26-java-thread-primitive-deprecation/)）。

### 死锁

废弃它们的首要原因是容易死锁：

```Java
This method has been deprecated, as it is inherently deadlock-prone.  

If the target thread holds a lock on the monitor protecting a critical 
system resource when it is suspended, no thread can access this resource
until the target thread is resumed. If the thread that would resume the
target thread attempts to lock this monitor prior to calling resume, 
deadlock results.  

Such deadlocks typically manifest themselves as "frozen" processes.
```

容易死锁的原因是 **被 `suspend` 挂起的线程不会释放锁**，因此若线程 t1 持有锁 L 时通过 `suspend` 挂起，线程 t2 `resume` t1 **之前** 必须先获取锁 L，然后死锁：

```Java
private static class Consumer extends Thread {
    private final Object lock;

    public Consumer(Object lock) { this.lock = lock; }

    @Override
    public void run() {
        synchronized (lock) {
            System.out.println("before suspend");
            // 挂起本线程，并一直持有 lock 监视器锁
            Thread.currentThread().suspend();
            System.out.println("after suspend");
        }
    }
}

Consumer c1 = new Consumer(Consumer.class);
c1.start();

TimeUnit.SECONDS.sleep(1);
synchronized (Consumer.class) {  // 需获取 Consumer.class 监视器锁才能进入同步块
    System.out.println("before resume");
    c1.resume();
    System.out.println("after resume");
}
```

结果只输出 `before suspend`，明显死锁。

### 活跃性问题

第二个问题是 `resume` 必须在 `suspend` 之后调用，否则无法恢复挂起线程：

```Java
private static class Consumer extends Thread {
    private final Object lock;

    public Consumer(Object lock) { this.lock = lock; }

    @Override
    public void run() {
        try {  // suspend 前等 1s，保证 resume 先执行
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {}

        synchronized (lock) {
            System.out.println("before suspend");
            Thread.currentThread().suspend();
            System.out.println("after suspend");
        }
    }
}

Consumer c1 = new Consumer(Consumer.class);
c1.start();

synchronized (Consumer.class) {
    System.out.println("before resume");
    c1.resume();
    System.out.println("after resume");
}
```

上面 `resume` 先于 `suspend` 执行，输出结果为：

```Java
before resume
after resume
before suspend
```

虽然执行了 `resume`，但线程 t1 永远无法恢复，发生活跃性问题。

## `LockSupport` 的改进

针对原生 `suspend`/`resume` 的缺陷，`LockSupport` 有针对性的做了改进：

1. 提供超时版 `parkNanos`/`parkUntil`，避免永久死锁；
2. `park`/`unpark` 调用顺序不会导致活跃性问题；
3. 提供 blocker object，便于问题定位、工具监控；

### 恢复死锁

首先普通的 `park`/`unpark` 使用不当同样会造成死锁，存在与 `suspend`/`resume` 相同的问题，但 `LockSupport` 还提供了超时版 park，可以从死锁中解脱出来：

```Java
private static class Consumer extends Thread {
    private final Object lock;

    public Consumer(Object lock) { this.lock = lock; }

    @Override
    public void run() {
        synchronized (lock) {
            System.out.println("before suspend");
            LockSupport.parkNanos(1000000000);  // 1s 后从挂起状态自动恢复
            System.out.println("after suspend");
        }
    }
}
```

>`parkNanos` 在以下情况下都可以从挂起状态恢复：
>
>1. 其他线程 `unpark` 该线程；
>2. 其他线程 **中断** 该线程；
>3. 超时；
>4. 莫名其妙的原因；

输出：

```Java
before suspend
// 间隔 1s 左右
after suspend
before resume
after resume
```

### 避免活跃性问题

`unpark` 可以先于 `park` 执行，此时 `park` 立即返回，不再挂起线程：

```Java
private static class Consumer extends Thread {
    private final Object lock;

    public Consumer(Object lock) { this.lock = lock; }

    @Override
    public void run() {
        try {  // suspend 前等 1s，保证 resume 先执行
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {}

        synchronized (lock) {
            System.out.println("before suspend");
            LockSupport.park();  // 2. 后执行
            System.out.println("after suspend");
        }
    }
}

Consumer c1 = new Consumer(Consumer.class);
c1.start();

synchronized (Consumer.class) {
    System.out.println("before resume");
    LockSupport.unpark(c1);  // 1. 先执行
    System.out.println("after resume");
}
```

输出：

```Java
before resume
after resume
before suspend
after suspend
```

### blocker 对象

从 Java 1.6 开始，`park`/`parkNanos`/`parkUntil` 都提供指定 blocker 对象的版本，也推荐大家尽量提供 blocker 对象，JUC 中的并发类，就全部使用带 blocker 对象版本的 `park`。

很多并发工具底层都使用了 `park`，如果没有 blocker，很难区分是哪个并发工具导致了线程挂起：

```Java
private static class Consumer extends Thread {
    private final int counter;

    public Consumer(int counter) { this.counter = counter; }

    @Override
    public void run() {
        if (counter < 10) LockSupport.park();
        System.out.println("counter = " + counter);
    }
}

Consumer c1 = new Consumer(5);
c1.start(); c1.join();
```

从线程转储看，只知道是 `LockSupport.park` 导致的线程挂起，没有其他信息：

```Java
 "Thread-0" #10 prio=5 os_prio=31 tid=0x00007f885f875000 nid=0x4403 waiting on condition [0x00007000070d3000]
    java.lang.Thread.State: WAITING (parking)
     at sun.misc.Unsafe.park(Native Method)
     at java.util.concurrent.locks.LockSupport.park(LockSupport.java:304)
     at Test$Consumer.run(Test.java:19)
```

如果加上 blocker 对象：

```Java
if (counter < 10) LockSupport.park("small");
```

线程转储就能提供更多信息：

```Java
 "Thread-0" #10 prio=5 os_prio=31 tid=0x00007fd0580c3800 nid=0x4003 waiting on condition [0x000070000c2b6000]
    java.lang.Thread.State: WAITING (parking)
     at sun.misc.Unsafe.park(Native Method)
     - parking to wait for  <0x0000000795773418> (a java.lang.String)
     at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
     at Test$Consumer.run(Test.java:19)
```

新增 `- parking to wait for  <0x0000000795773418> (a java.lang.String)` 这行，说明 blocker 对象是 `String` 类型。





