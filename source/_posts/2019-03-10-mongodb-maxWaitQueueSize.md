---
title: MongoDB | 如何配置连接池？
date: 2019-03-10 10:34:39
tags:
  - MongoDB
  - 连接池
categories: 数据库
---

最近有位同事遇到一个 MongoDB 的线上异常，类似：

```
Exception in thread "pool-1-thread-301" com.mongodb.MongoWaitQueueFullException: Too many threads are already waiting for a connection. Max number of threads (maxWaitQueueSize) of 500 has been exceeded.
    at com.mongodb.PooledConnectionProvider.get(PooledConnectionProvider.java:70)
    ...
```

异常原因堆栈里说的很清楚：

```
Too many threads are already waiting for a connection. 
Max number of threads (maxWaitQueueSize) of 500 has been exceeded.
```

<!-- more -->

即请求数据库连接的线程太多，已经超过 **等待队列** 的最大值，所以请求被直接 **拒绝** 了。

最无脑的解决方式是增加队列最大长度，连接池是客户端行为，不同驱动的实现可能不同，在 [Java 驱动](http://mongodb.github.io/mongo-java-driver/3.10/javadoc/?com/mongodb/MongoClientURI.html) 中，相关的配置有：

* `maxPoolSize`：连接池中的最大连接数量；
* `waitQueueMultiple`：该配置 * `maxPoolSize`  => `maxWaitQueueSize`，计算出等待连接的线程队列的最大长度，若超出限制，则抛出异常，丢弃该线程；
* `waitQueueTimeoutMS`：暂无空闲连接时，允许线程等待的最长时间（ms）；

所以，无法直接设置 `maxWaitQueueSize`，只能通过 `maxPoolSize` 和 `waitQueueMultiple` 两个参数间接设置：

```Java
connectionPoolSettings = ConnectionPoolSettings.builder()
    ...
    .maxWaitQueueSize(getThreadsAllowedToBlockForConnectionMultiplier() * getConnectionsPerHost())
    ...
    .build();
```

`connectionsPerHost` 对应 `maxPoolSize`，即线程池大小，默认 100，`threadsAllowedToBlockForConnectionMultiplier` 对应 `waitQueueMultiple`，默认 5，因此 `maxWaitQueueSize` 默认大小为 500（100 * 5），要增加等待队列长度，增大 `maxPoolSize` 或者 `waitQueueMultiple` 即可。

单纯避免这个异常很简单，但问题是该如何决定这些配置的合理值呢？

下面将分别介绍如何确定连接池、等待队列、连接超时等 3 个配置的合理值。

## 连接池

Little's Law 是排队论中的一个公式：

```
L = λ * W
```

在一个稳定系统中，顾客数量 `L` 等于顾客到达速率 `λ` 乘以顾客逗留时间 `W`，其中 `L`、`λ`、`W` 都是长期平均值，与顾客到达时间和顺序无关。

Little's Law 应用非常广泛，对应到请求排队的场景：

* `L` 即平均使用的连接数；
* `λ` 即系统吞吐量，单位为 “请求/s”；
* `W` 即每个请求的处理时间；

因此如果我们知道 `λ` 和 `W`，就能推导出一直保持使用的平均连接数，该值即为 **连接池** 的合理值。

假设 `λ = 500`，即每秒需要处理 500 个请求，`W = 30ms`，即每个请求数据库需要 30ms 才能响应，则平均使用的数据库连接数 `L` 为：

```
L = λ * W
  = 500 请求/s * 0.03 s/请求
  = 15
```

因此，当前负载（`λ = 500 请求/s`）和数据库处理能力（`W = 0.03 s/请求`）下，系统会保持使用 15 个数据库连接，因此连接池大小应为 15，从而最大化连接池的利用率。

>注意：500 和 30ms 都是本例中的假设值，生产环境通过测试获取。

## 等待队列

前面借助 Little's Law 计算得到“平均每时每刻有 15 个数据库连接请求”，但这只是平均值，系统实际负载不断变化，请求数量的峰值可能远远超过 15（想象一下双十一），若没有等待队列，洪峰来临时会有 **大量请求被拒绝**，因此等待队列必不可少。

首先，等待队列有以下作用：

1. 削峰填谷，防止高负载下的大量请求被拒绝；
2. 限制请求的 **最大延迟**；

削峰填谷容易理解，但凡队列都有该作用，可等待队列是如何影响延迟时间的呢？

假设等待队列长度为 500，且请求的平均处理时间为 30ms，则每个请求的最大延迟为：

```
最大延迟
  = 最大排队时间 + 该请求处理时间
  = 500 * 0.03 s + 0.03 s
  = 15.03 s
```

因此等待队列越长，则最大延迟越大，如果要限制延迟上限，则可以减小等待队列的长度，这与直觉相符。

因此等待队列长度需要考虑：

1. 请求洪峰有多大？
2. 能容忍的最大延迟有多大？

这两个因素互相影响，一起决定等待队列的长度，假设洪峰请求数量为 100 个，如果要成功处理这 100 个请求，则队列长度应为 100，此时最大延迟为 3.03s（100 * 0.03s + 0.03s），该延迟超过大多数用户的忍耐上限，一般用户可能在 1s 后没看到响应就会关闭页面，因此我们想要更短的延迟，所以需要减小等待队列，比如到 40，此时最大延迟为 1.23s，大约可用。

## 超时时间

超时时间也是连接池的重要配置，给定如下配置，连接超时应该设置多少呢？

* 连接池大小 15
* 等待队列长度 40
* 请求平均处理时间 30ms

超时时间与 **最大延迟** 相关，因此应该设为 40 * 0.03s + 0.03s，即 1.23s 左右，

一个常见误区是根据 **平均延迟** 来设置超时，例如等待队列大部分时间并不满，若大部分时间等待队列只有不到 10 个请求排队，则平均延迟肯定大大低于 1.23s，比如可能是 500ms，如果把超时时间设置为 500ms，则会导致队列较满时，很多排队的请求超时失败。





