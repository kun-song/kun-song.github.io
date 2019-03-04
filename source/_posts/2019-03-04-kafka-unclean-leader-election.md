---
title: Kafka | Unclean Leader Election
date: 2019-03-04 20:00:47
tags:
  - Kafka
categories: 技术
---

Kafka 对消息存储可靠性的保证是：

>A committed message will not be lost, as long as there is **at least one** in sync replica alive, at all times.

Kafka 当然可靠，但可靠是有前提的，即 ISR（in-sync replicas）至少存在一个副本，若无法满足该前提，则 Kafka 无法保证消息存储的可靠性。

<!-- more -->

ISR 不为空是 Kafka 系统模型的 **假设**，但系统实际运行中肯定会出现 ISR 为空的场景，比如 ISR 只有一个 leader replica，该 leader 所在 broker 又崩溃了，此时 ISR 为空。

实际的分布式系统需要处理 **所有** 可能出现的故障，包括 ISR 为空，针对 ISR 为空有两种处理策略：

1. 等待 ISR 集合中的 replica 恢复，并将它选为 leader，未恢复这段时间该 partition 不可用；
2. 将 out-of-sync replica 选为新 leader，保证了可用性，但可能造成消息丢失，（consumer 读取的）消息不一致等问题；

第一种策略被称为 clean election，第二种策略被称为 unclean election，它们在 **可用性** 和 **一致性** 之间做出了某种权衡。

若允许 out-of-sync replica 成为 leader，则可能 **丢失** 已写入原 leader，但还没有同步到 out-of-sync replica 的消息，且不同 consumer 读取到的消息可能 **不一致**（有的从原 leader 读，有的从新 leader 读）。

若不允许，则因为没有 ISR 供选举，所以只能等原 leader 被恢复后，该 partition 才能重新上线，期间该 partition 无法读、写。

通过 `unclean.leader.election.enable` 选择不同策略，Kafka 0.11.0.0 之后，默认为 `false`，即禁止 unclean election。

实际使用时，需要根据业务要求来决定是否禁止 unclean election，比如银行交易业务肯定要禁止，因为我们认为此时一致性需求远大于可用性，银行宁可停止服务半天，也不愿意出现有问题的交易；而日志收集业务则可以开启 unclean election，因为此时可用性更重要，几条日志丢失并不是什么大问题。


