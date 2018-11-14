---
title: 基本概念 | Round Robin
date: 2018-11-14 21:56:52
tags:
  - Akka
  - Kafka
  - DNS
categories: 技术
---

平时看英文文档的时候，经常遇到 round robin 的说法，比如：

* Akka 中的 [RoundRobinPool 和 RoundRobinGroup](https://doc.akka.io/docs/akka/current/routing.html#roundrobinpool-and-roundrobingroup)
* Kafka 中的 [roundrobin 分区策略](https://kafka.apache.org/documentation/)
* [Round-robin DNS](https://en.wikipedia.org/wiki/Round-robin_DNS)

从字面意思上可以猜测，round robin 有“轮流”、“轮换”的含义，如果只是为了表达轮流，直接用 round 不就完了，为啥非要用 round robin 呢？

<!-- more -->

今天专门看了下[维基百科](https://en.wikipedia.org/wiki/Round-robin_tournament)，原来 round robin 是一个惯用语：

>该术语来源于含义为“带子”的法语词 ruban，久而被讹用并成为惯用语。
>
>在17、18世纪时法国农民希望以请愿的方式抗议国王时，通常君主的反应是将请愿书中最前面的两至三人逮捕并处决，所以很自然地没有人希望自己的名字被列在前面。为了对付这种专制的报复，人们在请愿书底部把名字签成一个圈（如同一条环状的带子），这样就找不出打头的人，于是只能对所有参与者进行同样的惩罚。这个办法后来也被英国皇家海军的水手们使用过。

round robin 源于法语，robin 本来是法语中的 ruban，对应英文中的 ribbon，即“带，绶带”的意思，round ribbon 即“环带”，联系上面的解释，当使用 round robin 时，指参与者以完全平等、轮流的方式参与某活动，各方执行次数完全相等，即：

>In a single round-robin schedule, each participant plays every other participant once. 
>
>If each participant plays all others twice, this is frequently called a double round-robin. The term is rarely used when all participants play one another more than twice, and is never used when one participant plays others an unequal number of times (as is the case in almost all of the major United States professional sports leagues – see AFL (1940–41) and All-America Football Conference for exceptions).

了解 round robin 的背景后，再遇到相关的文档就更容易理解了。
