---
title: 读书 | 《Kafka 权威指南》
date: 2019-02-19 20:03:52
tags:
  - Kafka
categories: 读书
---

一句话评价：非常干，没有一句废话！

本书的作者阵容非常强大，有 Kafka 的作者、经验丰富的架构师，还有专家级的 SRE，让本书既深入代码原理，又兼顾介绍 Kafka 在整个数据系统中的作用，还提供了运维 Kafka 的宝贵经验。

为什么说本书干货满满呢，拿第一章举例，一般书籍的简介部分会比较水，但本书不然，第一章主题很多：

<!-- more -->

* Kafka 基本概念
  + 消息、批次
  + 模式
  + 主题、分区
  + 生产者、消费者
  + broker 集群
  + 多集群
* Kafka 优势
  + 多生产者、多消费者
  + 磁盘存储
  + 伸缩性强
  + 高性能
* 数据生态系统

每个主题虽然着墨较少，但信息量却很大，例如介绍模式只用了两段：

>While messages are opaque byte arrays to Kafka itself, it is recommended that additional structure, or schema, be imposed on the message content so that it can be easily understood. There are many options available for message schema, depending on your application’s individual needs. Simplistic systems, such as Javascript Object Notation (JSON) and Extensible Markup Language (XML), are easy to use and human-readable. However, they lack features such as robust type handling and compatibility between schema versions. Many Kafka developers favor the use of Apache Avro, which is a serialization framework originally developed for Hadoop. Avro provides a compact serialization format; schemas that are separate from the message payloads and that do not require code to be generated when they change; and strong data typing and schema evolution, with both backward and forward compatibility.
>
>A consistent data format is important in Kafka, as it allows writing and reading messages to be decoupled. When these tasks are tightly coupled, applications that subscribe to messages must be updated to handle the new data format, in parallel with the old format. Only then can the applications that publish the messages be updated to utilize the new format. By using well-defined schemas and storing them in a common repository, the messages in Kafka can be understood without coordination. Schemas and serialization are covered in more detail in Chapter 3.

读完这两段，我知道了 Kafka 消息以字节数组在网络上传输，但字节数组难以理解，所以最好为消息定义模式（schema），至于模式定义有很多可选方案，如：

* JSON/XML 
* Apache Avro/Protocol Buffer 等

它们之间的区别在于是否携带强类型信息，以及版本兼容性。

最后作者推荐使用集中式的模式管理，以避免生产者、消费者的直接耦合。

看到了吧，作者把模式相关的很多知识都告诉你了，只有短短的两段，却并不简单。

要理解模式如何携带类型信息，以及模式之间的兼容性到底是什么，我推荐你阅读 [Designing Data-Intensive Applications](https://book.douban.com/subject/26197294/) 的第四章 Encoding and Evolution，你想知道的所有关于序列化以及模式的内容都在那里。

再比如第二章，有的书籍会长篇大论外加无数截图介绍如何将 Kafka 安装到笔记本上，本书关于这点只有区区不到两页。

剩下篇幅介绍如何从磁盘吞吐量、磁盘容量、内存、网络、CPU 等角度为生产环境 Kafka 选择硬件，Kafka 因为自身设计原因，对磁盘、网络要求较高，而对内存、CPU 要求较低，本书会告诉你 Kafka 为什么是这样。

生产环境一般以 Kafka 集群方式运行，那问题来了，如何推算集群容量？如何配置操作系统（虚拟内存、磁盘、网络）以使 Kafka 性能最大化？JVM 又该如何配置，应该使用何种垃圾收集器？机架是否会影响集群的可用性？所有问题都在第二章得到解答。

从前两章可以看到，本书是道硬菜 :)

Kafka 有 4 个主要的使用场景，分别在如下章节介绍：

* 消息队列（第三、四章）
* 数据管道（第七章）
* 跨集群数据镜像（第八章）
* 流式处理（第十一章）

除此之外，有两章（九、十章）专门介绍运维，尤其是第十章，提供了监控 Kafka 的很多真知灼见。

第五章无关具体使用，深入介绍了 Kafka 的基本实现/原理，如控制器、复制、存储等，第六章介绍如何 producer、broker、consumer 三者如何协同实现可靠的数据传递，我认为这两章属于本书的高阶内容，与平时使用距离稍微远一点。

如果你觉得本身过于晦涩，读起来一头雾水，我推荐你先读下 [I Heart Logs](https://book.douban.com/subject/26107374/) 这本小册子，这本书只有六十页，作者 Jay Kreps 也是 Kafka 的作者之一，它介绍了以日志为核心的系统设计思路，算是 Kafka 的导读吧。

最后，关于使用 Kafka Streams 进行流式处理，最近出了一本专门的书：[Kafka Streams in Action](https://www.goodreads.com/book/show/34921247-kafka-streams-in-action)，评价也很好，值得一读。

