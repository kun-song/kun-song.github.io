---
title: Java NIO epoll bug 以及 Netty 的解决之道
date: 2019-07-26 23:04:28
tags:
  - Netty
categories: Java
---

最近学习 Netty 时，看到 Netty 宣称解决了很多 Java 原生 NIO 的很多 bug，其中之一是 epoll 空轮询导致 CPU 利用率 100%。

<!-- more -->

## 什么是 epoll 空轮询

如果使用 Java 原生 NIO 来编写服务器应用，代码一般类似：

```Java
// 创建、配置 ServerSocketChannel
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.socket().bind(new InetSocketAddress(9998));
serverChannel.configureBlocking(false);

// 创建 Selector
Selector selector = Selector.open();

// 注册
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // select 可能在无就绪事件时异常返回！

    Set<SelectionKey> readyKeys = selector.selectedKeys();
    Iterator<SelectionKey> it = readyKeys.iterator();

    while (it.hasNext()) {
        SelectionKey key = it.next();
        ...  // 处理事件
        it.remove();
    }
}
```

`selector.select()` 应该 **一直阻塞**，直到有就绪事件到达，但很遗憾，由于 Java NIO 实现上存在 bug，`select()` 可能在 **没有** 任何就绪事件的情况下返回，从而导致 `while(true)` 被不断执行，最后导致某个 CPU 核心的利用率飙升到 100%，这就是臭名昭著的 Java NIO 的 epoll bug。

>实际上，这是 Linux 系统下 poll/epoll 实现导致的 bug，但 Java NIO 并未完善处理它，所以也可以说是 Java NIO 的 bug。
>
>该问题最早在 Java 6 发现，随后很多版本声称解决了该问题，但实际上只是降低了该 bug 的出现频率，起码从网上搜索看，Java 8 还是存在该问题（[当 Thrift 遇到 JDK Epoll Bug](https://juejin.im/entry/5995084ef265da247779d23c)）。

## Netty 的解决之道

很多 NIO 框架都在 Java 原生 NIO 基础上增加了解决 epoll 空轮询的增强，本文介绍 Netty 的做法。

Netty 的解决方式分为两步：

1. 检测 epoll bug；
2. 通过重建 `Selector` 解决 epoll bug；

其实大部分框架的解决方式都类似，差别仅在 **检测方式**，检测到后基本都是通过重建 `Selector` 来解决。

### 检测 epoll 空轮询

Netty 使用 `NioEventLoop.select()` 替代 `Selector.select()`，检测 epoll bug 的逻辑就在 `NioEventLoop.select()` 中：

```Java
selectCnt = 0;  // epoll 空轮询场景下 select 调用次数
long currentTimeNanos = System.nanoTime();  // 每个 for 循环开始时的绝对时间

for (;;) {
    timeoutMillis = ...  // 初始化超时参数

    int selectedKeys = selector.select(timeoutMillis);
    selectCnt++;

    long time = System.nanoTime();  // 记录执行到此处的绝对时间：

    // 检测逻辑
    if (time - currentTimeNanos > timeoutMillis) {
        selectCnt = 1;  // 未发生 epoll 空轮询，所以把 selectCnt 重置为 1
    } else if (selectCnt >= 重试次数阈值（默认 512）) {
        selector = selectRebuildSelector(selectCnt);  // 解决 epoll bug 的实际逻辑
        selectCnt = 1;  // 解决本次 epoll bug，重置 selectCnt
        break;
    }

    currentTimeNanos = time;  // 重置下次 for 循环开始时间
}
```

如果满足以下两个条件，则认为发生 epoll 空轮询：

1. `selector.select(timeoutMillis)` 阻塞时间小于 `timeoutMillis`，且
2. `select` 执行次数 > 阈值（默认 512）

因为阻塞时间无法做到很精准，所以若某次阻塞时间大于等于 `timeoutMillis` 立刻重置 `selectCnt` 为 1，即需要 **连续** 512 次 `selector.select(timeoutMillis)` 阻塞时间都小于 `timeoutMillis` 才认为发生了 epoll 空轮询。

`timeoutMillis` 有一套计算逻辑，无法进行配置，而次数阈值可以通过 `io.netty.selectorAutoRebuildThreshold` 系统配置进行设置，默认值为 512。

### 解决 epoll 空轮询

检测到 epoll bug 后，通过 `selectRebuildSelector` 方法来实际解决：

```Java
private Selector selectRebuildSelector(int selectCnt) throws IOException {
    // The selector returned prematurely many times in a row.
    // Rebuild the selector to work around the problem.
    logger.warn("Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.", selectCnt, selector);

    rebuildSelector();  // 重建逻辑
    Selector selector = this.selector;

    // Select again to populate selectedKeys.
    selector.selectNow();
    return selector;
}
```

重建过程代理给了 `rebuildSelector` 方法，重建完成后，立即 `selectNow` 重新监听事件。

而 `rebuildSelector` 又把重建逻辑代理给了 `rebuildSelector0`：

```Java
/**
 * Replaces the current Selector of this event loop with newly created Selectors to work
 * around the **infamous epoll 100% CPU bug**.
 */
public void rebuildSelector() {
    if (!inEventLoop()) {
        execute(new Runnable() {
            @Override
            public void run() {
                rebuildSelector0();  // 重建逻辑
            }
        });
        return;
    }
    rebuildSelector0();  // 重建逻辑
}
```

`inEventLoop()` 判断当前线程是否是事件循环线程：

```Java
@Override
public boolean inEventLoop() {
    return inEventLoop(Thread.currentThread());
}
```

而 `inEventLoop(Thread)` 定义在 `EventExecutor` 中，不同实现类的实现逻辑不同，`NioEventLoop.inEventLoop` 具体实现在其父类 `SingleThreadEventLoop` 中： 

```Java
@Override
public boolean inEventLoop(Thread thread) {
    return thread == this.thread;
}
```

在 Netty 中，一个 IO 线程可以处理多个 channel，但一个 channel 只能被一个 IO 线程处理，重建 `Selector` **必须** 在事件循环线程内完成，如果当前线程是 `NioEventLoop` 线程，则直接在当前线程执行 `Selector` 重建，否则将重建任务 `submit` 给各个 `NioEventLoop`。

添加该判断的原因是除了 `NioEventLoop` 检测到 epoll bug 时会调用 `rebuildSelector` 外，`NioEventLoopGroup` 也有调用：

```Java
public void rebuildSelectors() {
    for (EventExecutor e: this) {
        ((NioEventLoop) e).rebuildSelector();
    }
}
```

该方法被暴露出来，供用户调用，因此最终 `rebuildSelector` 是可能在非事件循环线程中被调用的。

重建任务最终在 `rebuildSelector0` 中完成，重建步骤：

1. 新建一个 `Selector`；
2. 将旧 `Selector` 的所有 channel 注册到新 `Selector` 上；
3. 关闭旧 `Selector`；

至此，完成对 epoll bug 的解决。

---

参考：

1. [Netty 解决/规避了 JDK 的 NIO bug 了吗？](https://www.zhihu.com/question/291370310)
2. [NIO 的 epoll 空轮询 bug](https://www.cnblogs.com/JAYIT/p/8241634.html)
