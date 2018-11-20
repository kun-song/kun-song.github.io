---
title: 译 | KIP62 Allow consumer to send heartbeats from a background thread
date: 2018-11-15 08:32:30
tags:
  - Kafka
categories: 技术
---

原文在 [这里](https://cwiki.apache.org/confluence/display/KAFKA/KIP-62%3A+Allow+consumer+to+send+heartbeats+from+a+background+thread)。

## 状态

本提案已经被接受，邮件讨论在 [这里](https://markmail.org/message/oeg63goh3ed3qdap)，本修改在 Kafka 0.10.1.0 首次发布。

## 动机

新 Kafka consumer 有个很蛋疼的设计问题：

* consumer 是 **单线程**，同时
* consumer 需要向 coordinator 发送 **心跳** 以保持活跃；

<!-- more -->

我们推荐用户在 **同一线程** 中执行：

* `poll` 循环
* 分区初始化/清理
* 消息处理

但若分区初始化/清理 + 消息处理时间超过 session timeout，则 consumer 将被从 consumer group 中移除，它负责的分区也会被分配到 group 中的其他 consumer，其影响如下：

* 最好的情况下，只会导致不必要的 rebalance 和 consumer regoin group；
* 最坏的情况下，consumer 将无法提交 **耗时较长** 消息的偏移量，导致 rebalance 后这些消息被不断重复处理，然后再超时，再 rebalance，再重复处理，极端情况下 consumer 将无法向前运行，直到用户察觉并修复该问题；

目前（0.10.1.0 之前）有两种解决方式：

1. 增加 `session.timeout.ms`，以提供更多消息处理时间；
2. 降低 `max.poll.records`，以减少每次 `poll` 返回的消息数量；

KIP-41 新增的 `max.poll.records` 解决了导致会话超时的一种场景，在此之前，无法限制单次 `poll()` 调用返回的 **消息数量**，因此对消息处理时长的调优只能基于 `max.partition.fetch.bytes`。在 KIP-41 之后，用户可以把 `max.poll.records` 设置的 **非常小**，从而不需要增大 `session.timeout.ms`，最终达到不牺牲 failure detection 时间的目标。

但 `max.poll.records` 只在 **消息之间** 处理时长变化不大时才有效，同时 consumer 一般采用批量消费模式，例如 Kafka Connect 就推荐对 sink connector 采用批量消费模式，因为其性能更好。

有时无法直接控制 `max.poll.records`，比如 Kafka Stream 封装了 Kafka Consumer 客户端，用户根本无法设置该参数，此时只剩下 `session.timeout.ms` 一条路了，增大超时时间虽然能解决问题，但 **牺牲** 了 broker 检测客户端失败所需时间，最后的最后，超时时间应该设置多大呢？只能不断在生产环境测试，然后调整超时时间，陷入测试--设置的痛苦循环。

批处理模式与 group rebalance 一起出现时尤为蛋疼，rebalance 开始后，consumer 必须在 session timeout 前完成：

1. 消息处理逻辑
2. 提交 offset
3. rejoin 群组

因此 session timeout 实际也是 rebalance timeout，若 consumer rejoin 地不够快（超过 session timeout 后未完成），则 consumer 将被从群组中移除，即使 consumer **仍然不断向 coordinator 发送心跳**！

因为 consumer 无法预知何时发生 rebalance，因此 consumer 发现 rebalance 时手头通常还有 **一批** 消息需要处理，没有啥有效策略处理该场景，大部分客户端仅仅是完成消息处理，如果提交 offset 时已经超时，则直接抛出异常。

poll 循环时的设计思想是，提供一种内置机制，以确保 consumer：

* alive
* make progress

只要 consumer 不断发送心跳，它就一直持有自己负责分区的 **锁**，但若 consumer：

* 无法正常处理消息
* 但仍然正常发送心跳

则群组中的其他 consumer 无法 **接管** 出问题 consumer 的分区，造成 lag 不断增长。

若心跳发送、消除处理处于同一线程中，则可以保证消息处理必须足够快，consumer 才能继续持有自己的分区，任何影响消息处理的因素同样会影响心跳发送，从而避免了活锁（livelock），即 consumer 没挂，但无法继续处理消息。

## 建议修改

我们认为以上问题的根源是 session timeout 同时用于：

* app 崩溃、不可达的标识
* consumer 处理一批消息的最大耗时
* rebalance 完成的最大耗时

本提案明确区分以上 3 种超时。

### 解耦消息处理超时

我们建议新增：

1. “消息处理超时”，即 `max.poll.interval.ms`；
2. 一个后台线程，在“处理超时”时间内保持 session active；

`max.poll.interval.ms` 是 consumer 两次调用 `poll()` 之间的 **最大延迟时间**，若两次 `poll()` 调用间隔超过该值，consumer 将停止发送心跳，并显式发送 `LeaveGroup` 请求，consumer 下次调用 `poll()` 时将 rejoin 群组，这与之前通过 `session.timeout.ms` 控制时 **完全相同**，唯一的区别是“处理超时”与“会话超时”已经解耦，因此可以：

* 设置较大的 `max.poll.interval.ms` 以适应耗时的消息处理；
* 设置较小的 `session.timeout.ms` 以实现快速 crash detection；

目前实现中，rebalance 和 fetch 仍然在调用 `poll()` 的线程中执行，后台线程只负责发送心跳，以及处理超时时发送 `LeaveGroup` 请求，可通过同步访问 `ConsumerNetworkClient` 实现该功能。

实践中，推荐 process timeout 至少与 session timeout 一样大，若发现“处理超时”较小，则需减少“会话超时”，将其设置为“处理超时”的值，因为实际上达到“处理超时”的效果与达到“会话超时”完全相同，为与现有 client 库兼容，我们选择“会话超时”、“处理超时”两者之间的 **较大值** 作为实际的处理超时时间，因此当“会话超时”作为实际处理超时时，后台线程不是必要的，以后可以作为一个优化点。

### 解耦 rebalance 超时

另外，我们建议解耦 session 超时和 rebalance 超时。目前，当 group 开始 rebalance 时，我们挂起每个 consumer 的 individual heartbeat timer，并启动 rebalance timer，rebalance timer 的超时时间被设置为 `session.timeout.ms`，我们建议增加独立的 rebalance timeout，并包含在 `JoinGroup` 请求中发送给 coordinator，这会导致增加 `JoinGroup` 的版本号。与 session timeout 相同，broker 将使用群组中最大的 rebalance timeout 作为最终的超时时间（一般群组内的 rebalance timeout 相同，除非正在滚动升级）。

rebalance timeout 应该从哪里获取呢？

`max.poll.interval.ms` 是消息处理的最大时间，它同样也是 consumer rejoin 群组的最大时间，因此我们建议将 rebalance timeout 设置为 `max.poll.interval.ms` 的值。rebalance 开始后，后台线程依然不断发送心跳，当 consumer 处理完消息，再次调用 `poll()` 时将 rejoin 群组，从 coordinator 角度看，consumer 只会在两种情况下被移除：

1. 未收到心跳的时间超过 session timeout
2. rebalance timeout expires

关于 rebalance timeout 还有几点值得注意：

* 解耦 session timeout 和 rebalance timeout 后，可更优雅的处理 hard crash
  * consumer hard crash 后，无法发送 `LeaveGroup` 请求，coordinator 只能依赖 session timeout 检测失败，若此时正好发生 rebalance，coordinator 必须在 rebalance 结束之前等待 session timeout 指定的时间；
  * 解耦后，通过定义非常小的 session timeout 可快速检测失败，而 rebalance 也能更快完成；
* 通过把 process timeout 用作 rebalance timeout，用户可优化 rebalance 对群组的影响
  * 例如，若要保证 rebalance 在 5s 内完成，则用户需要保证每批消息在 5s 内处理完，以前通过 session timeout 也能达到该效果，但需要权衡错误检测时间；
  * process timeout 更小，则 rebalance 可更快完成，但 commit failure 概率也会变大，因为消息处理完成之前 consumer 可能被踢出群组；

## 公共接口

增加 `max.poll.interval.ms` 后，可以将 session timeout 设置地的非常小，以快速检测错误（目前将 session timeout 设为 30s 的唯一原因是保障消息处理可完成），为避免大部分用户手动调整，我们提供以下默认值：

* `session.timeout.ms`: 10s
* `max.poll.intevals.ms`: 5m
* `max.poll.records`: 500

我们减小了默认 session timeout（30s -> 10s），同时通过 process timeout 增加了消息处理时间，以前 `max.poll.records` 默认值为 `Integer.MAX_VALUE`，现在将其设为更合理的 500。

本提案在 `JoinGroup` 中新增 `RebalanceTimeout`，因此其版本号将增加为 1：

```
JoinGroupRequest => GroupId SessionTimeout RebalanceTimeout MemberId ProtocolType GroupProtocols
  GroupId => string
  SessionTimeout => int32
  RebalanceTimeout => int32  ;;; this is new
  MemberId => string
  ProtocolType => string
  GroupProtocols => [ProtocolName ProtocolMetadata]
    ProtocolName => string
    ProtocolMetadata => bytes
```

## 兼容性、废弃与迁移方案

从技术角度看，这些变动与现有客户端不兼容，主要问题是 session timeout 语义上的变化，现在 session timeout 仅仅控制检测 consumer 崩溃的时间。

* 使用默认配置的用户可能根本察觉不到该变化，即使察觉，也是好的变化：
  * consumer 错误检测时间更快（session timeout 默认值由 30s -> 10s）
  * 消息处理时间上限更大（session timeout 30s -> process timeout 5m）
* 若用户增大了 session timeout 以提供更多消息处理时间，因为我们使用 session timeout 和 process timeout 之间的较大值作为实际的 process timeout，所以若：
  * 用户自定义 session timeout <= 5m，则实际处理时限变为 5m
  * 用户自定义 session timeout > 5m，则实际处理时限不变，还是 session timeout
* 若用户减小了 session timeout，新增的 process timeout 将导致活锁检测时间变长
  * 本来用 session timeout 即可检测活锁，现在需要用 process timeout，其默认值 5m，明显更长
  * 我们认为用户不会强依赖该检测时间，因此应该问题不大

因此，即使本 KIP 不向前兼容，但影响不大，同时新 broker 和老 client 也能共存，当 broker 收到版本为 0 的 `JoinGroup`请求时，coordinator 将使用 session timeout 作为 rebalance timeout，与之前行为相同。
