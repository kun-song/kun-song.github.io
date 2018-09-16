---
title: MongoDB | ObjectId 生成
date: 2018-09-16 21:31:01
tags:
  - MongoDB
categories: 技术
---

在 MongoDB 中每个 document 都有一个 `_id` 字段，该字段用于标识（通常是唯一标识）collection 中的特定 document。

`_id` 字段可以是任何类型，但默认类型为 `ObjectId`，`ObjectId` 是一个 BSON 类型，长度为 12 字节，且：

* 4 字节：时间戳
* 3 字节：机器标识码
* 2 字节：进程 ID（pid）
* 3 字节随机数/自增计数器

<!-- more -->

我们知道很多数据库的主键是在 server 端生成的，由于大部分传统数据库节点不多，因此可以比较容易地在 **不同节点** 之间同步 id 的值，保持 id 在所有节点中的一致性。

MongoDB 是天生的分布式数据库，可以有成百上千节点，节点甚至可以分布在不同数据中心，因此在 server 生成 id 难度很大，所以 MongoDB 选择在 client 端由 drvier 生成 id。

通过综合考虑时间、机器、进程 3 个因素，再加上随机数，可以基本保证 id 的唯一性。

当然万事无绝对，重复 id 确实有可能生成。

首先，ObjectId 最后 3 字节为计数器时，当在 1s 内（因此前 4 字节的时间戳相同），在同一机器的同一进程中，生成文档数量大于 2^24，则必然存在重复 id。

其次，某些 driver 将最后 3 位实现为随机数，则不需要大于 2^24 个文档就可能出现重复。

最后，不同机器、进程可能产生相同的标识码，该场景下，若两个进程在同 1s 内，产生了两个相同的计数器值，则也会导致重复 id。

当然第 1 和 3 几乎不可能出现，而且目前大部分 drvier 不会用随机数作为 ObjectId 的后 3 字节，因此重复 id 产生概率是非常小的。



---

参考：

* [Possibility of duplicate Mongo ObjectId's being generated in two different collections?](https://stackoverflow.com/questions/4677237/possibility-of-duplicate-mongo-objectids-being-generated-in-two-different-colle/10183273)
