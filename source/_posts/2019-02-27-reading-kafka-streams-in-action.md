---
title: 读书 | Kafka Streams in Action
date: 2019-02-27 23:15:05
tags:
  - Kafka
  - Kafka Streams
categories: 读书
---

[豆瓣链接](https://book.douban.com/subject/30314195/)

因为工作中要用 Kafka Streams，所以看完《Kafka 权威指南》后马不停蹄把这本书看了下。

之前我对流式处理的了解基本停留在理论上，看过几本相关的书，如 [DDIA](https://book.douban.com/subject/26197294/) 和 [Making Sense of Stream Processing](https://book.douban.com/subject/30324756/)，但是从来没实际写过代码，这次终于用上了。

<!-- more -->

本书作者 Bejeck 也是 Kafka 社区的长期贡献者，对框架本身非常了解，虽然主题关于 Kafka Streams，但内容尽量保持了通用，很多概念和技巧可用于其他框架，比如聚合、时间窗口、事件时间与处理时间的区别、连接操作（stream-table、table-stream、stream-stream）、stream 和 table 的关系等，抛开实现细节不谈，这些内容在各框架之间基本是通用的，因此不光能学到 API 的使用，也能了解通用知识，这是好书的一个特点，从这个角度看，这本书还不错。

Kafka Streams 是基于 Kafka consumer/producer API 实现的，因此学习 Kafka Streams 能加深对 Kafka 本身的理解，说实话，个人觉得 Kafka Streams 实现方式很有意思，也很巧妙，它把可靠性、可扩展性、容错等完全建立在 Kafka 能力上，大量使用了 Kafka 的各项特性，比如：

* 通过 compact log 实现 state store
* 通过 batch 减少网络传输
* 通过 partition 实现 task 并行处理
* 通过 Kafka replication 实现 state 容错
...

当然，我现在对 Kafka Streams 了解还很浅薄，更多细节现在还看不出来。

如果你只需要使用 Stream DSL，那么只看第 3、4、5 以及第 7 章（监控）和第 8 章（测试）即可，如果要用 `Processor` API，再加第 6 章，其余是辅助章节。

至于书的难度，因为我之前已经过了一遍官方文档，而且对流式处理和 Kafka 都有所了解，所以读起来比较顺畅，大约一周时间就看完了，个人建议阅读之前了解下 Kafka 为好。

缺点嘛，书是 2018 年 7 月出版的，到现在半年左右，然而 Kafka Streams 已经从 1.x 升级到 2.x，部分 API 有变化，而作者维护的实例代码没有持续升级，还是有点恼火，建议有兴趣的同学抓紧看，现在还基本在保质期内。

最后谈下对 Kafka Streams 的浅薄理解，我们知道内存、硬盘、网络之间的速度差别巨大（大约各差 1000 倍），当年 Spark 对 MapReduce 的改进之一就是减少不必要的网络操作，尽量使用内存操作。而 Kafka Streams 不可避免涉及大量（多大？）网络操作，从这个角度看，它注定不是一个高效的框架。

那为什么要设计它呢，官方文档也说了，使用 Spark Streaming 等有很大的运维开销（搭建、监控、运维、调优 Spark 集群），而且需要将任务代码提交到 Spark 集群，这与普通的应用部署形态不同，而且部分流式处理很轻量级，借助 Kafka Streams 可以直接将原来的普通应用修改为流式引用，马上可以享受流式处理的优势，又避免了额外的运维成本，这种说法还是有点道理的，像 Pulsar Function 也是类似的思路，每个框架都有自己合适的使用场景、设计取舍，我们根据场景合理选择即可。
