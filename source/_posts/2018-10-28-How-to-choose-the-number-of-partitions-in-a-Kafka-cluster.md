---
title: 译 | How to choose the number of partitions in a Kafka cluster?
date: 2018-10-28 23:44:39
tags:
  - Kafka
categories: 技术
---

[原文链接](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster)

很多 Kafka 用户都有同样的疑问，本文将讲解释分区数量选择的几个重要影响因素，并提供简单的计算公式。

## 分区越多，吞吐量越高

### 原理

首先需要明确，主题分区（topic partition）是 Kafka 中的 **并行单元**（the unit of parallelism）。

* 在 producer 端和 broker 端，对不同分区的写操作可以 **完全并行执行**，因此耗时操作（如压缩）可以更好地利用硬件资源；
* 在 consumer 端，Kafka 会把单个分区分配给一个 consumer 线程（即分区与 consumer 是 1 : N 的关系），因此 consumer 端的并行度上限受（该 consumer group 消费的）分区数量限制。

所以，总体而言，Kafka 集群中分区越多，则可达的吞吐量越高。

<!-- more -->

### 估算

可基于吞吐量简单计算所需分区数量。

假设经过测试，对于单个分区，producer 端吞吐量为 p，consumer 端吞吐量为 c，若系统目标吞吐量为 t，则至少需要 `max(t/p, t/c)` 个分区。

单个分区的 produce/consume 吞吐量的影响因素如下：

* producer：由 batching size、压缩编码、acknowledgement 类型、复制因子等配置决定；
* consumer：由应用处理消息的业务逻辑决定；

根据经验，单个分区 produce 吞吐量在几十 M 级别，但 consume 吞吐量必须通过测试获取。

### 注意

虽然可以随时增加分区数量，但要特别注意消息中携带 key 的场景。

当写入该类型的消息时，Kafka 根据 hash of key 决定该消息应该存放的分区，因此 key 相同的消息总是存储在 **同一分区** 上，且分区内消息是 total order 的，部分应用的正确性依赖该 **顺序保证**（例如 Change Data Capture 中，write 消息乱序将导致数据不一致）。

若分区数量变化，可能打破该保证（相同 key 存储于同一分区，进而相同 key 的消息保持全序），从而导致 consumer 业务错误。

为避免该问题，常见的做法是多分配一些分区（over-partition a bit），即根据对未来一到两年系统吞吐量的预期来决定分区数量。业务初始阶段可以根据当前吞吐量使用小规模的 Kafka 集群（每个 broker 容纳较多分区），业务增长过程中可以逐渐增加新 broker，并将原来 broker 上的分区在线迁移到新 broker 上，从而即可以保证 key 的读取顺序，又能适应不断增长的业务要求。

除吞吐量外，选择分区数量时还需要考虑其他因素，分区过多也可能带来负面影响。

## 分区越多，文件描述符越多

每个分区映射到 broker 文件系统中一个目录，每个目录中，对于每个 segment 有两个文件（索引 & 实际数据），目前 Kafka 为每个 segment 的 index 和 data 文件各打开一个文件描述符。

因此分区越多，则文件描述符的配置上限需要越大，通常这只是个配置问题，我们见过每个 broker 打开 3w 多个文件的线上 Kafka 集群。

## 分区越多，可能影响可用性

Kafka 支持分片复制，每个分片可能有多个 replica，不同 replica 可以部署在不同 broker 上，分片复制是 single-leader 架构，即其中一个 replica 被选为 leader，其余都是 follower。

producer 和 consumer 请求都通过 leader partition 执行，broker 发生故障时，该 broker 上 leader 所在 partition 复制集暂时不可用（因为请求必须由 leader partition 处理），此时一个被称为 controller 的特殊 broker 会将不可用 partition 复制集的 leader 转移到其他 broker 的 replica 上，从而完成故障转移。该过程中，controller 会读写 Zookeeper 上保存的分片元数据，且读写 **线性** 执行。

若 broker 安全关闭（参考 Kafka Graceful shutdown），controller 可主动执行 leadership transfer，每个 leader 转移仅需几毫秒，因此客户端能感知的不可用窗口非常小。

然而，若 broker 非安全关闭（例如 `kill -9`），客户端感知的不可用窗口与 **分区数量** 成正比。假设某 broker 有 2000 个分区，每个分区有两个 replica，若 leader 在 broker 间均匀分布，则该 broker 上大约有 1000 个 leader，若该 broker 发生故障，则这 1000 个 leader 几乎同时不可用，Kafka 会逐一为每个 partition 副本集选举新 leader（因为 controller 对 Zookeeper 的读写线性执行），若每次选举需 5ms，则总共需要 5s 才能完成 1000 次选主，因此对某些 partition 而言，客户端不可用时间为 5s + 检测该 broker 不可用所需时间。

更坏场景下，若故障的 broker 恰好是 controller，则 Kafka 首先要完成 controller 的故障转移，然后才能为 partition 副本集选主。controller 故障转移是自动执行的，但新 controller 需从 Zookeeper 读取所有 partition 的元数据，以便完成初始化，若 Kafka 集群中有 10000 个 partition，每个 partition 读取需要 2ms，则总共需要 20s 才能完成新 controller 初始化。

通常而言，unclean failure 并不常见，但如果关心这种极端场景下的可用性，则最好将每个 broker 的 partition 数量限制在 2000-4000 个，整个集群 partition 数量限制在小几万（low tens of thousand）。

## 分区越多，E2E 时延可能越大

Kafka 将 E2E 时延（end-to-end latency）定义为 producer 发布消息与 consumer 成功读取消息之间的时间。

producer 发布消息到对应 partition 复制集的 leader 上，然后被 **复制** 到所有 partition replica 上后，Kafka 才 commit 该消息。因此 message commit 时间在 E2E 时延中可能占较大比例，默认配置下，broker 使用 **单线程** 完成 broker 之间 **全部 partition** 的复制，在我们的实验中，从一个 broker 复制 1000 个 partition 到另一个 broker 需要 20ms，因此 E2E 时延至少是 2ms，对于实时性要求很高的应用而言 20ms 太高了。

注意，大规模集群会缓解 partition 个数对时延的影响，假设 broker A 有 1000 个 leader partition：

* 若集群中只有两个 broker，则 commit a message 耗时为几十毫秒；
* 若集群中还有 10 个 broker，则平均而言，每个 broker 仅需从 broker A 上取 100 个 partition 的数据，因此 commit a message 耗时将从几十毫秒降低到几毫秒；

因此，如果很关心时延，最好将每个 broker 上的 partition 数量限制在 `100 * b * r` 以下，其中：

* `b` 为 Kafka 集群中的 broker 数量；
* `r` 为复制因子；

## 分区越多，客户端可能占用更多内存

我们随 Kafka 0.8.2 发布了 Confluent Platform 1.0，其中包含更加高效的 Java producer 实现，该实现可以设置 producer 使用的内存上限，其中有个实现细节为：producer 会为每个 partition 缓存消息，当消息总量或缓存时间到达一定界限后，producer 才清空缓存，并把消息发送到 broker。

若 partition 数量增加，则 producer 需要缓存更多 partition 的消息，若缓存消息的总内存超出配置的内存上限，则 producer 要么阻塞，要么丢弃新消息，两种行为皆不可取，为避免该场景，需要重新设置更大的内存上限。

一般而言，为得到良好的吞吐量，应为每个 partition 的缓存消息预留几十 KB 内存，并根据 partition 总数计算 producer 的内存使用上限。

consumer 也存在同样的问题，consumer 批量获取每个 partition 的消息，所以更多 partition 意味着更多内存占用，不过 consumer 一般处理速度非常快（接近实时），所以这在 consumer 端一般不是事。

## 总结

总之，partition 越多，则吞吐量越大，但过多 partition 也会带来负面影响，如：

* 影响可用性
* 增加时延

等。
