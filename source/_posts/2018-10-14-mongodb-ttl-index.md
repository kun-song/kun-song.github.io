---
title: MongoDB | TTL 索引
date: 2018-10-14 21:13:09
tags:
  - MongoDB
  - 索引
  - TTL
categories: 数据库
---

MongoDB 有几种很实用的特殊索引，在特定场景中可以达到事半功倍的效果，TTL 索引即其中之一。

过期特定数据是一种常见场景，对 IoT 设备日志、网站 session 等数据而言，我们一般只关心一定时间内的数据，过期数据需要及时删除，TTL 索引很适合用于该场景。

<!-- more -->

## 索引创建

TTL 索引是特殊的 **单字段** 索引，且字段类型必须是：

* BSON [date](https://docs.mongodb.com/manual/reference/bson-types/#document-bson-type-date)，或
* BSON date 数组

创建时指定 `expireAfterSeconds` 属性即可：

```
createIndex( { "lastModifiedDate": 1 }, { expireAfterSeconds: 60 * 60 } )
```

* `lastModifiedDate` 类型为 BSON [date](https://docs.mongodb.com/manual/reference/bson-types/#document-bson-type-date)；
* `lastModifiedDate` 单位为秒；

每个文档的 `lastModifiedDate` 时间点加 `expireAfterSeconds` 指定的时间段即为该文档的 **过期时间点**，到达该时间点后，TTL 索引将文档标记为过期，并由专用的后台 TTL 线程定期检查、删除（默认 60s 运行一次）。

若 TTL 索引字段为 date 数组，则 MongoDB 选择其中 **最早** 的时间来计算过期时间。

注意以下场景中，文档永远无法过期：

* TTL 字段不是 date/date 数组；
* 某文档不包含 TTL 字段；

## 文档删除

索引创建方式会影响 TTL 线程删除过期文档的时间：

* 后台方式（`{background: true}`）：创建过程中，即开始删除；
* 前台方式（`{background: false}`）：创建完成后，才开始删除；

TTL 线程每 60s 执行一次删除动作，并且删除动作耗时受 MongoDB 实例负载影响，因此文档过期与文档被删除之间存在 **不定时延**，可能立即被删除，也可能超过 60s 后才被删除。

若 MongoDB 以复制集方式部署，则 TTL 线程仅在 [主节点](https://docs.mongodb.com/manual/reference/glossary/#term-primary) 上运行删除任务，[复制节点](https://docs.mongodb.com/manual/reference/glossary/#term-secondary) 通过 **复制** 主节点的删除操作完成删除。

## 注意事项

* TTL 索引是单字段索引，复合索引将 **忽略** `expireAfterSeconds` 属性；
* `_id` 字段不支持 TTL 索引；
* MongoDB 无法删除 [capped 集合](https://docs.mongodb.com/manual/core/capped-collections/) 中的文档，因此无法在 capped 集合上创建 TTL 索引；
* [collMod](https://docs.mongodb.com/manual/reference/command/collMod/#dbcmd.collMod) 命令加 [index](https://docs.mongodb.com/manual/reference/command/collMod/#index) 标志可以修改已有 TTL 索引的 `expireAfterSeconds` 属性值；
* 若字段 `A` 上已经有一个非 TTL 的索引，则只能先 drop 该索引，然后才能在 `A` 上创建 TTL 索引；

------

参考：

* [MongoDB TTL index 文档](https://docs.mongodb.com/manual/core/index-ttl/)
