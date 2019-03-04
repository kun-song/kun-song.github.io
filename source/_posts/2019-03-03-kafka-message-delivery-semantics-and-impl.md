---
title: Kafka 消息语义
date: 2019-03-03 17:18:35
tags:
  - Kafka
  - 消息语义
categories: 技术
---

常见的消息投递语义（message delivery semantics）有 3 种：

* At most once
  + 消息可能丢失，但不会重复投递
* At least once
  + 消息不丢失，但可能重复投递
* Exactly once
  + 不丢失、不重复

不同语义的 **实现成本不同**，虽然 exactly once 是最完美的语义，但它的实现成本最高。

<!-- more -->

在 Kafka 中，消息传递链路为：

```
producer 发送消息 -> broker 集群持久化消息 -> consumer 消费消息
```

以上 3 种语义需要从发送、持久化、消费 3 个阶段端到端进行保证。

Kafka 没有一个配置参数来指定系统整体的消息投递语义，而是通过 producer 和 consumer 的多个可配置行为来 **组合** 实现特定的语义，下面分别分析 producer 和 consumer 影响消息投递语义的配置和用法，最后给出要实现指定语义，producer 和 consumer 应该如何配置和使用。

Kafka 的错误模型（fault model）明确系统可能出现：

* 网络暂时故障：
  + producer 发送失败
  + consumer 消费失败
* broker 出现磁盘故障、节点失效
* consumer 崩溃

因此 Kafka 保证即使出现以上故障，也能保证满足消息投递语义。

## producer 对消息语义的影响

producer 发送消息后等待 broker 响应该消息是否 commit，但 producer 可能无法收到响应消息，原因可能有：

1. 由于网络故障，broker 没有接收到消息；
2. 由于 broker 故障（如磁盘满、未达到最小 ISR 数量、partition leader 崩溃等），未能成功 commit 接收到的消息；
3. broker 持久化成功，但由于网络故障，producer 未能接收到响应；

此时 producer 没有选择，只能重新发送消息，producer 对投递语义的影响主要是 **重发消息**：

* 在 0.11.0.0 之前，重新发送消息可能（例如，上面第 3 条）造成 broker 中存储重复消息，对应 at least once；
* 在 0.11.0.0 之后，Kafka 实现了消息发送的 **幂等性**，因此重新发送不会造成 broker 中存在重复消息，对应 at most once & exactly once。

>* 发送幂等实现：每个 producer 具备唯一 ID，每条消息有一个单调的序列号，broker 根据 ID 和序列号对消息进行 **去重**；
>* 0.11.0.0 还实现了发送到多个主题消息的事务操作，以帮助实现 exactly once 语义；

## consumer 对消息语义的影响

consumer 的处理流程可以抽象成 3 个步骤：

1. 接收消息
2. 处理消息
3. 提交消息 offset

第一步肯定是接收消息，但后面两个步骤却可以交换位置，因此有两种组合：

1. 接收消息 -> 处理消息 -> 提交 offset
2. 接收消息 -> 提交 offset -> 处理消息

第一种组合，若处理消息时 consumer 崩溃，则此前已经处理的消息会被其他 consumer 重复处理，对应 at least once 语义；第二种组合，若处理消息时 consumer 崩溃，则会丢失部分消息，对应 at most once 语义。

consumer 如何实现 exactly once 语义呢？

主要有两种方式，第一种类似 producer 以 **幂等** 方式重复发送消息，consumer 可通过消息的 key 进行去重，实现幂等，进而达到 exactly once 效果。

第二种需借助”事务“，前面说过，producer 可将多个发送到不同主题的动作放到一个“事务”中，对于 consumer 而言，它的处理输出分为两部分：

* offset
* 消息处理结果

如果将这两部分数据存放在不同主题上，并通过“事务”保证其原子性，则可实现 exactly once 语义。

根据事务隔离级别不同，该行为表现也不同：

* read_uncommitted
  + 若事务被终止，则 consumer 可以观察到 offset log 和 output log 中的中间状态
* read committed
  + 不可以观察被终止事务的中间状态

>通过 `isolation.level` 配置 Kafka 事务的隔离级别，有两个可选值：
>* `read_uncommitted`（默认）
>  + `poll()` 可以返回未提交的事务性消息，也可能返回已经 abort 的消息；
>* `read_committed`
>  + `poll()` 仅返回已提交的事务性消息；
>
>注意，无论哪种隔离级别，非事务性消息都不受该配置影响。

若消息处理结果本来就要发送到 Kafka 某主题上，自然可以使用上面的方式，但有时候，我们需要把处理结果持久化到其他存储，比如关系型数据库上，此时怎么办呢？

例如，offset 存放在 Kafka topic 上，而处理结果需存放在 MySQL 上，如何实现 exactly once 呢？

一种方式是通过分布式事务，如两阶段提交，但有的数据库不支持。

另一种方式是把 offset 也存储到 MySQL 中，借助 MySQL 事务，可以保证 offset 和处理结果的原子性。

## 不同语义对应的配置

### at least once

producer 默认幂等重发，consumer 遵循“接收 -> 处理 -> 提交”，则实现 at least once 语义。

### at most once

取消 producer 自动重发，consumer 遵循“接收 -> 提交 -> 处理”，则实现 at most once 语义。

### exactly-once

若消息处理结果也发送到 Kafka 上，则借助“事务”可实现 exactly once 语义，例如 Kafka Streams 底层实现。

若消息处理结果发送到其他存储上，则有两种方式实现 exactly once 语义：

* 分布式事务
* 消息处理结果 + offset 一起存放到其他存储，利用该存储的事务实现

>offset 默认实现就是存放到 Kafka 主题上。
