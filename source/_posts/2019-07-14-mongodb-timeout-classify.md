---
title: MongoDB 超时配置
date: 2019-07-14 10:27:02
tags:
  - MongoDB
categories: 数据库
---

MongoDB 有很多超时配置，虽然它们都有各自的默认值，但为了提升应用的及时响应性、性能等，有时需根据实际情况进行修改。

>不同语言、同一语言不同版本的驱动，对同一配置可能采用不同的默认值，细节需要查阅文档。

以下是比较常用的 3 个：

1. 服务器选择超时；
2. 连接超时；
3. Socket 超时；

<!-- more -->

## 服务器选择超时

MongoDB 是分布式数据库，根据配置不同，可能是单机，也可能是具备多分片、多副本的集群，客户端进行读写请求时都要选择具体的服务器：

1. 写操作需要选取 primary 服务器；
2. 读操作既可以选取 primary，也可能选取 secondary，也可以根据地理位置选取，无论如何，需要根据一定算法选一个服务器出来；

Java 驱动的默认值为 30,000ms，如果无法进行服务器选择，默认等待 30s 才会报错，该超时主要根据业务要求进行设置。

## 连接超时

选取服务器完成后，下一步开始建立与该服务器的 Socket 连接，影响连接建立快慢的因素有：

1. 客户端与服务器之间的网络质量；
2. 服务器的工作负载，如果服务器连接达到最大值，则需要排队等待；
3. 服务器可能挂了、进程被杀死等；

Java 驱动的默认值为 10,000ms，如果服务器早就挂了，客户端也得傻等 10s 才报错，该超时主要通过对生产环境测试得出。

## Socket 超时

Socket 连接建立成功后，下一步是在该连接上 **发送请求**、**接收响应**，

Java 驱动默认值为 0，即无超时，请求或响应将被 **无期限地等待**，对大多数应用而言，这明显不合理，所以使用时至少需要修改 Socket 超时。

---

参考：

1. [Server Selection Algorithm](https://docs.mongodb.com/manual/core/read-preference-mechanics/)
2. [MongoClientOptions 文档](https://mongodb.github.io/mongo-java-driver/3.4/javadoc/com/mongodb/MongoClientOptions.html)