---
title: OLTP vs OLAP
date: 2018-10-03 12:48:27
tags:
  - OLAP
  - OLTP
categories: 技术
---

OLTP 和 OLAP 是数据库的两种应用场景：

* OLTP: online transaction processing
* OLAP: online analytic processing

先有 OLTP 后有 OLAP，下面简单介绍两者的由来和特点。

<!-- more -->

## OLTP

数据库最早用于商业数据处理，那时写入操作常常对应一个商业交易（commercial transaction）。

交易处理要求低延迟的读写，且每次只处理全部记录中的 **一小部分**，通常根据某些键来过滤需要的数据，例如根据用户 id 之类的。

交易系统需要根据用户输入插入、更新记录，因为有 insert/update/read，所以整个系统是交互式的，因此这种访问模式被称为 online transaction processing，即 OLTP。

虽然后来数据库使用场景越来越广泛，不再局限于金融应用，但 OLTP 的访问模式却与以前非常类似。

## OLAP

随着应用越来越复杂，公司内部会有数据分析的需求，数据分析的访问模式与 OLTP 差别很大，每次需要加载 **大量** 记录进内存（且常常仅需要每条记录的少量字段），然后对它们进行统计分析，统计结果将用于公司管理或商业决策（business intelligence）。

为了与 OLTP 区别开，该访问模式被称为 online analytic processing，即 OLAP。

最开始大家直接在 OLTP 数据库上做分析，这样做有几个缺点：

* OLAP 访问模式与 OLTP 差别很大，直接在 OLTP 系统上作分析并不方便，比如数据加载速度很慢；
* OLTP 系统一般是公司的盈利应用，需要保持低延迟、高可靠性，分析操作可能导致数据库访问速度快速下降，影响用户体验；
* 公司发展壮大后，会有很多 OLTP 系统，分析操作最好能在全部数据上进行，直接在 OLTP 系统上分析，局限性很大；

后来一般用独立的适合 OLAP 的数据库进行分析操作，并通过 ETL 将数据从 OLTP 系统中加载到数据仓库（warehouse）中。

## 比较

OLTP 系统一般面向 **最终用户**，需要处理 **大量请求**（每个用户对应一个请求），因此存储引擎一般借助 index 加速查找操作，此时系统瓶颈常常是 **硬盘查找时间**。

OLAP 系统一般面向公司内部的商业分析师，其请求数量很少，但每个请求需要加载大量数据，此时系统瓶颈常常是 **硬盘带宽**。

下面是更加详细的比较：

* 读模式
  * OLTP：每次请求的记录数量很少
  * OLAP：大量记录
* 写模式
  * OLTP：随机访问，数据来源是用户输入，要求低延迟
  * OLAP：批量导致（ETL）、事件流
* 使用场景
  * OLTP：面向最终用户的 web 应用
  * OLAP：内部分析师，用于决策
* 数据量
  * OLTP：GB/TB
  * OLAP：TB/PB

其实不仅数据库层会区分 OLTP 和 OLAP，底层的存储硬件也会针对不同访问模式进行优化，比如华为的存储就可以配置是用于 OLTP 还是 OLAP。

----

参考：

* [Designing Data-Intensive Applications](https://book.douban.com/subject/26197294/)
