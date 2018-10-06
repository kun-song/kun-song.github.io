---
title: 读书 | Making Sense of Stream Processing
date: 2018-10-02 22:31:50
tags:
  - 日志
  - Kafka
  - stream processing
  - Event Sourcing
  - CQRS
categories: 读书
---

豆瓣：[Making Sense of Stream Processing](https://book.douban.com/subject/30324756/)

前段时间读了 [I heart Logs](https://book.douban.com/subject/26107374/) 这本小册子，感觉意犹未尽，于是跟随 [阿莱克西斯](https://www.zhihu.com/people/ming-zi-zong-shi-hen-nan-qi/activities) 的脚步，又看了下这本书。

本书作者还有一本更广为人知的书 [Designing Data-Intensive Applications](https://book.douban.com/subject/26197294/)，关于 DDIA 的评价可以参考 [2017 年，你看了啥很好的计算机的博客/书/视频](https://www.zhihu.com/question/263874795/answer/274182358)，作者的技术水平、写作态度还是很值得信赖的。

<!-- more -->

其实这三本书有千丝万缕的联系，尤其是前两本，都是介绍以日志为中心的系统架构，但侧重点不同：

* I heart logs 是对日志架构模式的基本介绍，类似入门手册；
* Making Sense of Stream Processing 偏重将日志架构应用于实际（流式）系统构建，实践性更强，内容更丰富；
  
而 DDIA 主题更为宏大，不局限于流式系统，而是对现代数据系统构建的方面面都有涉及，尤其是对分布式系统的讲解很赞。

本书一共五章，个人认为每章内容都很赞。

第一章将各领域关于事件流的术语统一介绍了一下，比如：

* Event Sourcing
* Reactive
* Complex Event Processing
* Actors
* Change Capture
* Stream Processing

相信有很多人跟我一样，听说过其中的部分名词，而且也对它们之间的关系很疑惑，第一章将解开该疑惑。

第二章介绍“纯粹的”基于日志的系统架构，以及该架构相对传统的以关系型数据库为中的架构的优势。

当然作者并非怂恿大家把已有的系统都推翻重来，对于既存应用，第三章介绍如何借助 Kafka 和 CDC（change data capture）将日志架构融入老系统中。

第四章很有意思，作者提出利用 Kafka 将 Unix 哲学应用于系统构建中，Unix 哲学内容很多，作者指的是这两条：

* Make each program do one thing well.
* Expect the output of every program to become the input to another, as yet unknowm program.

简单说就是各司其职，做小而精的工具，通过组合实现大而全的应用，熟悉 Unix/Linux 的同学对此应该很熟悉，平时用到的 awk、sed、grep 等都是这样设计的。

Unix 工具之间的组合性通过统一接口实现，它们的输入都是 stdin，输出都是 stdout/stderr，自然非常容易组合。

这种设计哲学经过了时间和实践的考验，是一种有效的系统构建方式，但条条大道通罗马，传统的关系型数据库选择了相反的设计方式（数据库集成了大量的特性，是大而全的典型），在过去的几十年里也取得了巨大成功。

遗憾的是在现代应用中，单一数据库越来越难以满足需求了，因此出现了很多各具特色的数据存储系统：

* Redis/Memcached：高速缓存
* Elastic Search：全文检索
* MongoDB：文档型数据库
* Neo4J：图数据库
* Hadoop：批量处理
* Spark Streaming/Flink：流式处理

大家的应用中很大概率会存在其中部分组件，应用现在是跑起来了，但是不同存储组件之间的数据集成、流动却不是那么容易的，作者在提出用 Kafka 也可以作为不同数据系统之间的“统一接口”，这样各种数据存储系统也可以灵活组合了，实际上很多应用都也是这么做的。

最后一章是作者对未来数据库和数据应用发展的一个展望，作者认为不同特性将从数据库中剖离出来，形成独立组件，组件之间可灵活组合，数据应用可以灵活选用需要的组件，这正是目前实际正在发生的。

通过本书，能跳出传统系统构建的条条框框，站在一个更新的角度看待我们的应用，多了一种系统构建的思路，非常不错。
