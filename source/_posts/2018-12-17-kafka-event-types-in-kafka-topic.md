---
title: 译 | 不同类型的事件可以放到同一主题中吗？
date: 2018-12-17 23:35:25
tags:
  - Kafka
categories: 技术
---

原文地址在 [这里](http://martin.kleppmann.com/2018/01/18/event-types-in-kafka-topic.html)，InfoQ 已经翻译成中文，本文仅总结个人所得。

使用 Kafka 时，若有多种不同类型的事件，应该把它们放到同一个 topic 中，还是放到多个不同 topic 中呢？

首先，topic 的首要作用为：

>allow a consumer to specify which subset of messages it wants to consume.

<!-- more -->

考虑两种极端：

* 若只有一个 topic，则消费者只能消费全部事件，无法自由选择感兴趣的事件子集；
* 若 topic 太多，如几百万，由于每个 topic 至少有一个分区，所以集群至少有几百万分区，将严重影响性能；

过多 topic 对性能的影响的主要原因是分区数量，更多关于分区数量对 Kafka 性能影响的分析，可以参考 [译 | How to choose the number of partitions in a Kafka cluster?](./2018-10-28-How-to-choose-the-number-of-partitions-in-a-Kafka-cluster.md)。

几千个 topic 会明显影响性能，若性能要求很高，可以将多个 **细粒度** 事件合并成一个 **粗粒度** 事件，从而减小分区数量对性能的影响。

## Topic = collection of events of the same type?

常见的推荐用法是：

* 把相同类型的事件放到同一 topic 中；
* 把不同类型的事件放到不同 topic 中；

这与关系型数据库有点类似，因为表（table）也是一组类型相同的记录（record）的集合。

[Confluent Avro Schema Register](https://docs.confluent.io/current/schema-registry/docs/index.html) 加强了这种观点，因为它鼓励用户对 topic 内的事件使用相同的 Avro 模式。

对部分应用（如用户行为日志）而言，要求同一 topic 内的事件模式相同（即类型相同 or 兼容）是合理的。

但对于将 Kafka 用作数据库的应用（如[event sourcing](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)、[exchanging data between microservices](https://www.confluent.io/blog/build-services-backbone-events/)）而言，topic 内的事件类型是否相同并不重要，它们更关心的是 Kafka 在每个 topic partition 内保持事件 **顺序** 的能力。

假设有一个 entity，例如 customer 实体，该 entity 上可以发生很多事件：

* 创建 customer
* 修改 customer 的地址
* 为 customer 添加信用卡
* customer 关闭自己的账号
* ...

这些事件的顺序非常重要：创建之前不能发生任何动作，关闭之后也不能再次添加信用卡。

若使用 Kafka 对其建模，可将所有 entity 相关的事件放到同一 partition，Kafka 将保证 partition 内事件的顺序，这时同一 topic 内出现了多种不同类型的事件。

## Ordering problems

若把 `customerCreated`、`customerAddressChanged`、`customerClosed` 等事件放到不同 topic 中，则消费者无法保证按照生产者的生产顺序来消费它们。

即使为每个事件绑定时间戳也不行。

首先，当处理完时间戳为 A 的事件后，无法确定 A 前面的事件都已处理完毕，此时要么等待一段随机时间，要么继续处理后面的事件，无论如何选择，都无法保证事件顺序。

其次，在分布式系统中，依赖时钟同步是非常麻烦的，参考 DDIA 第 8 章。

## When to split topics, when to combine?

有了前面的背景，本节给出何时放入同一 topic，何时拆分到不同 topic 的一般准则。

### 1. The most important rule is that any events that need to stay in a fixed order must go in the same topic and the same partition.

>全局无序，partition 内有序。

一般同一 entity 的不同事件才关心事件顺序，因此要把它们放到同一 topic 的同一 parition 上，如 event sourcing。

### 2. For events about different entities, it depends.

若 entity 之间有依赖关系，如 address 从属于 customer，或一组 entity 经常一起使用，则这些 entity 的事件最好放到同一 topic 中。

若各 entity 没有关联，且由不同团队负责，则最好放到不同 topic 中。

另外，还要考虑吞吐量，若某事件吞吐量远远超过其他事件，则最好把它拆分到独立 topic 中，不然只关心其他低频事件的消费者也必须消费该事件。

### 3. What if an event involves several entities?

例如 purchase 事件涉及 product 和 customer 两个实体，transfer 事件涉及转入、转出两个 account 实体。

推荐开始时将 event 记录为 single atomic message，不要过早将其拆分为（多个 topic 中的）多个事件。因为借助 stream processor 可轻易拆分 compound event，但重建原始事件却非常困难。

可以给原始事件一个 UUID，将该事件拆分为关于每个 entity 的事件时，携带该 UUID，从而实现事件追踪。

### 4. If several consumers all read a particular group of topics, then consider to combine them.

把细粒度 topic 合并为粗粒度 topic 后，部分 consumer 可能会收到并不感兴趣的事件，这不是大事，因为从 Kafka 消费事件非常轻量级，即使 consumer 选择忽略一半的事件，这种 overconsumption 消耗也不会很大。

只有当 consumer 需要忽略大部分事件（如 99.9%）时，才推荐将低频事件拆分出来。

### 5. A changelog topic for a Kafka Streams state store should be a separate from all other topics.

### 6. The last one you should choose.

若前面的准则都不适用，则把相同类型的事件放到同一 topic 中，该准则相当于什么都没说。

## Schema Management

事件编码的影响：

* 使用无模式的 JSON 编码时，同一 topic 中可以轻易存放很多不同类型的事件；
* 使用有模式的编码时，如 Avro，同一 topic 中还是可以存放不同类型的事件，但没那么容易；

前面说过，[Confluent Avro Schema Register](https://docs.confluent.io/current/schema-registry/docs/index.html) 基于一个 topic 只有一种模式的假设，模式（schema）可以修改，但要保持向前、向后兼容，这种设计的优点是：多个 producer/consumer 可同时使用不同的模式，且保持互相兼容。

在 Avro Schema Register 注册模式时，需要指定主题（subject name），默认主题为 topic-key/topic-value，Register 将为该 subject 下注册的模式检查其向前、向后兼容性。

作者为 Register 做了修改，以允许同一 topic 中出现多种类型的事件，具体可参考原文，现在 Confluent Avro Schema Register 也没有原来的限制了。

