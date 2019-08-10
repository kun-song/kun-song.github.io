---
title: ACID 含义浅析
date: 2019-08-10 17:17:18
tags:
  - 事务
categories: 数据库
---

事务的安全保证分为 4 类：

1. Atomicity
2. Consistency
3. Isolation
4. Durability

前面说过，不同数据实现的 ACID 可能差别很大，本文只总结 ACID 的通常含义，要理解特定数据库的 ACID 具体含义，需要查阅其文档。

<!-- more -->

## Atomicity

对于 **多个写操作** 组成的一个业务操作，若执行到中间时遇到错误，如：

1. app 进程崩溃；
2. DB 进程崩溃；
3. 网络中断；
4. DB 磁盘满；
5. ...

若无事务，则 app 需要自己记录已经执行的写操作，然后一个个手动回退，以消除部分更新，最后再重试，非常繁琐。

若有事务，则：

>If an error occurs halfway through a sequence of writes, the transaction should be **aborted**, and the writes made up to that point should be **discarded**. 
>
>In other words, the database saves you from having to worry about **partial failure**, by giving an all-or-nothing guarantee.

即中间发生错误的事务将：

1. **终止** 当前事务；
2. **丢弃** 当前事务中已经执行的更新；

app 只需重试即可，逻辑大大简化。

## Consistency

ACID 中的一致性与 CAP、一致性哈希、最终一致性中的一致性都不同，它是一种 app 自己定义的 **数据约束**，只有满足该约束的数据才 **合法**。

C 是 app 层面定义的逻辑约束，并非 DB 本身能自动提供的，需要 app 自己借助 DB 的某些特性来维持。

常见的一致性约束有：

1. 身份证 ID 必须保持唯一；
2. 年龄字段不能为负值；
3. 转账前后，账户数据必须收支平衡；

对于这些一致性约束，有的可以借助 DB 提供的约束实现，例如唯一性、不能为负值等；有的则需要借助事务的 AI 属性来实现，如最后一个。

## Isolation

>Isolation 解决并发问题。

与并发编程时类似，多个 app 可能同时访问、修改 DB 中的同一（行、文档、k-v）数据，如果不加控制，则来自不同 app 的读写操作序列可能 **互相交叉**，最终结果 **无法预测**，也无法重现，即发生 **竞态条件**。

假设两个 app 同时执行以下操作组成的事务：

1. 读取 id = 1 的行；
2. 将该行的 count 字段 + 1；
3. 将结果写回 DB；

若无并发控制，最后的结果无法预测，可能是 +1 也可能是 +2，这与并发编程时遇到的问题完全相同。

在多线程编程中，有多种解决方式：

1. 对共享变量（id = 1 的行）加锁，使多线程在该操作序列上 **串行化**，即 **悲观锁**；
2. 使用 CAS 操作，多线程在该操作序列上仍是 **并行**，但某线程 +1 成功后，其他线程 +1 将失败，需要 **重试**，最终所有线程都能 +1 成功，即 **乐观锁**；

在 DB 领域，也有对应的解决方式，即 Isolation：

>Concurrently executing transactions are isolated from each other, they can't step on each other's toes.

但怎么个隔离法要看具体实现了，ANSI SQL 定义了四种隔离级别：

1. read uncommitted;
2. read committed;
3. repeatble read;
4. serializability;

不同隔离级别能解决的问题、付出的代价各不相同。

最严格的是串行化，即并发执行的事务执行结果与它们 **串行执行** 时结果相同，整个 DB 是一个 **全局一致性副本**，各个 app 都在同一份数据上操作，**完全解决** 了竞态条件。

但串行化有很大的性能开销，很少商用数据库采用该隔离级别，例如 Oracle 使用 snapshot isolation 替代串行化。

## Durability

持久性保证

>若事务 **已提交**，则 DB 绝对不会丢失其更新数据，即使提交后 DB 进程崩溃 or 磁盘损坏。

对于单节点 DB，事务提交成功需满足：

1. 更新数据成功写入 **非易失性存储**，如磁盘、SSD 等（相对而言，内存为易失性存储，**掉电** 丢数据）；
2. 更新数据成功写入 WAL（write ahead log），保证非易失性存储存储上的数据损坏时，可以从 WAL **恢复** 数据；

对于分布式 DB，有额外要求：

1. 更新数据成功 **复制** 到足够节点（不同共识算法要求的副本数量不同）；

>单个节点复制成功的标准与单节点 DB 相同，即写非易失性存储 & WAL 成功。

只有复制到足够节点后，事务才能 **提交** 成功，Kafka 生产者写消息时与此类似，只有消息写入足够多 ISR（in-sync replica）时，broker 才响应消息写入成功。

注意，持久性无法保证 **任何场景** 都不丢失数据，如：

1. 单节点 DB：若数据本身和 WAL 同时损坏，则数据无法恢复；
2. 分布式 DB：若足够多节点崩溃，则共识算法无法对数据达成一致，DB 处于不一致状态，数据页无法恢复；
