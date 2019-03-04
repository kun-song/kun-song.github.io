---
title: Kafka offset 提交方式
date: 2019-03-03 14:21:03
tags:
  - Kafka
categories: 技术
---

Kafka 通过 `enable.auto.commit` 控制 offset 提交方式：

* `true`：通过后台线程周期性提交（默认）
* `false`：手动提交

## 自动提交

既然是周期性提交，就需要配置提交周期，Kafka 通过 `auto.commit.interval.ms` 配置提交周期，默认值是 5000ms。

<!-- more -->

大部分提交是定时任务，也有部分例外，比如 `pool()` 和 `close()` 也会提交当前的最大偏移量。

### 缺点

自动提交可能导致：

* 丢失消息
* 重复消费

若 `pool()` 返回消息后是异步处理，或同步处理耗时过长，则自动提交时，若消息没有处理完，也会将它们提交，在 Kafka broker 看来，这些消息已经完成处理。若此时 consumer 崩溃，则未处理的消息在当前 consumer 丢失，该 consumer 负责处理的分区将被负载均衡到其他 consumer，其他 consumer 将从最新偏移量开始消息，从而彻底丢失这部分消息。

再看下重复消费的场景，若 `pool()` 返回的消息处理了一部分，但由于时间原因，还未自动提交，此时若 consumer 崩溃，则其他 consumer 接手它的分区时，会从上次提交的偏移量处开始消费，从而再次拉取相同的消息，造成重复消费。

最后，自动提交时，**无法避免** 丢失消息、重复消费。

## 手动提交

为避免丢失消息、重复消费，可选择手动提交。

### 同步

`commitSync()` 将提交上次 `pool()` 返回（批量）消息中的最大 offset，且阻塞到 broker 提示提交成功，因此非常可靠，若提交失败，则抛出异常，详细解释提交失败的原因。

注意，同步提交也 **可能** 出现：

* 丢失消息
* 重复消费

因为 `commitSync()` 将提交最大 offset，所以一定要确保提交时所有消息已经处理完成，否则可能 **丢失消息**，例如先提交，再真正处理消息。

而若 `pool()` 返回后，`commitSync()` 提交前，发生 rebalance，则 `commitSync()` 将因 rebalance 提交失败，该批次消费将被其他 consumer 再次 `pool()` 获取，因此在该 `pool()` 和 rebalance 之间处理的消息将被 **重复消费**。

最后，使用 `commitSync()` 时，若保证提交时所有消息已完成处理，则可 **避免消息丢失**，但无法避免重复消费。

>`commitSync` 遇到提交失败时会 **自动重试**，大部分错误可通过重试解决，若重试（几次？）后遇到重试无法解决的错误，再抛出异常，因此不要捕获异常，然后自己 **手动重试**。

### 异步

`commitSync()` 将阻塞直到提交成功 or 失败，这增加了可靠性，但降低了吞吐量，因此 Kafka 提供了另一个选择 `commitAsync()`。

前面说过，同步提交会 **自动重试**，直到成功，或遇到重试无法解决的错误（如 rebalance 导致的提交失败），但异步提交不会重试，因为在异步提交中允许重试可能造成大量重复消费。

比如现在异步提交时的 offset 为 500，但因为网络原因该提交发送失败，该错误应该可以通过重试解决，因此 consumer 会重新提交 offset = 500，但因为是异步提交，所以重试时可能已经成功提交了 offset = 5000（记住，Kafka 可能每秒消费百万级消息），若允许重试，则会把当前 offset 重置为 500，导致后面大量消息被重复消费，所以在异步提交中应该禁止自动重试。

`commitAsync` 提供回调函数，供提交失败时调用，可以在回调中打印日志，也可以在回调中尝试重试，但如同前面所说，重试异步提交可能 **重置 offset**，从而造成重复消费，因此要特别小心。

可以通过一个单调增长的计数器解决重试异步提交的问题，具体做法是：

1. 每次调用 `commitAsync()` 时将计数器加 1，并将计数器当前值传入其回调函数；
2. 回调函数中检查计数器的当前值与提交时的传入值是否相等，若相等则再次提交，若当前值更大，说明有更新的 offset 被提交，则放弃重试；

伪代码如下：

```Java
val commitCount = 0

val callback = count ->  (offsets, e) -> if (commitCount == count)  consumer.commitAynsc()

consumer.commitAsync(callback(commitCount));
```

### 同步 + 异步

若因为暂时问题（如暂时的网络故障）导致提交失败，则没有重试机制影响不大，因为后续的其他提交将修复该问题。

但若该次提交是 consumer 停止之前的最后一次提交，我们希望能重试最后这个失败的提交，此时可结合使用同步和异步提交：

```Java
try {
  while (true) {
    ConsumerRecords records = consumer.pool()
    ... processing
    comsumer.commitAsync()
  }
catch (ex) {
  log.error()
}
finally {
  try {
    consumer.commitSync()
  }
  finally {
    consumer.close()
  }
}
```

正常场景下使用 `commitAsync()`，异步提交速度更快，而且若一直在 `while(true)` 中正常运行，则某次提交失败将被随后的其他提交修复，随后的这些提交可以视为某种“重试”。

若 consumer 终止，则 **最后一次提交** 使用 `commitSync()`，同步提交将 **一直重试**，直到成功，或遇到无法通过重试解决的错误。

通过结合同步、异步两种提交方式，综合了异步提交的高效和同步提交的可靠。

### 提交指定偏移量

`commitSync()` 和 `commitAsync()` 将提交上次 `pool()` 返回的最大偏移量，若每次拉取的消息批量太大，每次又仅提交最大偏移量，则若发生 rebalance，大量消息会被重复处理，此时可以在处理批量消息中间，多次少量提交。


