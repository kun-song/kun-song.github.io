---
title: JVM | 为何生产环境最好保持 Xms = Xmx ？
date: 2018-12-19 08:58:48
tags:
  - Xms
  - Xmx
categories: JVM
---

很早就听老司机说，在生产环境，最好把 Xms 和 Xmx 设为相同值，并且 [Oralce 官方]() 也是这样推荐的：

Setting initial and minimum heap size | Xms | Oracle recommends setting the minimum heap size (-Xms)equal to the maximum heap size (-Xmx) to minimize garbage collections.
-- | -- | --

这是为什么呢？

首先，Xms/Xmx 对 JVM 行为的影响如下：

<!-- more -->

>app 启动时，JVM 将申请大小为 Xms 的内存作为堆的初始内存，随着新对象不断创建，堆空闲部分越来越少，当空闲部分少于 2% （GC 也无法增加更多空闲时）时：
>* 若堆已经达到 Xmx，则抛出 `OutOfMemoryError`；
>* 若未达到 Xmx，则 JVM 向 OS 申请内存，增加堆大小；

因此，把 Xms/Xmx 设为相同值有如下优势：

1. 生产环境中，一般而言，app 启动很短时间后，JVM 堆大小就会增加到 Xmx，中间多次按需增加堆大小，每次增加需要向 OS 请求内存（耗时），且 JVM 一般 **不会释放** 已持有的内存，不如一步到位，以 Xmx 堆内存启动；
2. 起始堆内存太小（Xms），会导致启动初期频繁 GC，起始堆内存较大（Xmx）有助于减少 GC 次数；
3. 生产环境 OS 一般只运行一个 app，**不存在资源竞争**，可以直接把所需最大内存分配给 app；

测试环境最好与生产环境一致，因此也推荐把 Xms/Xmx 设为相同值。

开发环境有所不同：

* 首先开发环境 app 负载不高，因此堆内存一般不会超过 Xms，不像上面说的那样，JVM 堆内存最终到达 Xmx；
* 其次开发环境 OS 一般运行很多应用（IDE、Email、通信工具等），还可能部署多个 app，因此 app 之间存在 **资源竞争**，*总内存可能根本不够所有 app 满负载运行*，较小的 Xms 有助于协调 app 之间的资源竞争。

我认为 app 初上线时，可以设置较小的 Xms 和较大的 Xmx，因为此时可能并不清楚 app 最终需要多大的堆，经过一段时间监控，找到堆的合适大小后，再把 Xms Xmx 设为该值，这是一个不断 profiling 的过程。

---

参考：

* [Why to set -Xms and -Xmx to the same value?](https://developer.jboss.org/thread/149559?_sscc=t)
* [Is it good to set the max and min JVM heap size the same?](https://stackoverflow.com/questions/6862972/is-it-good-to-set-the-max-and-min-jvm-heap-size-the-same)
* [Why is it recommended to have Xms and Xmx(memory jvm arg) values be same in middleware application server?](https://www.quora.com/Why-is-it-recommended-to-have-Xms-and-Xmx-memory-jvm-arg-values-be-same-in-middleware-application-server)



