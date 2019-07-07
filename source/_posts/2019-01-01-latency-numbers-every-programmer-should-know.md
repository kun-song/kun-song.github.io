---
title: Latency Numbers Every Programmer Should Know
date: 2019-01-01 17:13:54
tags:
categories: 分布式
---

Jeff Dean 和 Peter Norvig 对计算机各类操作的耗时做了大致估计（我找到的最早出处是 Peter Norvig 著名的 [Teach Yourself Programming in Ten Years](http://norvig.com/21-days.html#answers)），具体数值随硬件的发展肯定有变化，伯克利有个[动态图表](https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html)，可查看每年各个操作耗时的变化，非常赞！

例如 2019 年数据如下：

<img src="/images/2019/latency-numbers/latency-numbers-2019.png" alt="latency numbers 2019" style="width: 800px;"/>

<!-- more -->

比较每年各个数值的变化，可发现一些有趣的现象。

第一，roundtrip in same datacenter 和 packet roundtrip CA to Netherlands 耗时没有任何变化，一致保持 500us 和 150ms，原因很好理解，毕竟信号在光纤中以近似光速传播，该时间由物理规律决定。

第二，从 2006 年开始，前两列操作的数值不再变化，只有后两列在变化，说明近十年来存储介质的速度有较大提升。

第三，SSD 的随机读取速度从 1990 年到 2019 年变化不同，不过从 19us 提升到 16us，但顺序读取速度却从 50ms 提升到 49us，提升巨大。

...

前面说过，具体数值不重要，重要的是对各操作之间耗时数量级的直观感受，为便于理解，把实际数值放大 10 亿倍，则：

* 秒级操作：

  ```
  L1 cache reference                   1  s
  Branch mispredict                    3  s
  L2 cache reference                   74 s
  Mutex lock/unlock                    17 s
  Send 2K bytes over commdity network  62 s
  ```

* 分钟级操作：

  ```
  Main memory reference               1.7 m
  Compress 1K bytes with Zippy        3.4 m
  ```

* 小时级操作：

  ```
  Read 1 MB sequentially from memory  1   hour
  SSD random read                     4.5 hour
  Read 1 MB sequentially from SSD     17  hour
  ```

* 天级操作：

  ```
  Round trip within same datacenter   6  day
  Read 1 MB sequentially from disk    11 day
  Disk seek                           35 day
  ```

* 年级操作：

  ```
  Packet roundtrip CA to Netherlands  4.76 year
  ```

这样一来，数量级的差别就很明显了，L1 cache reference 相当于一次心跳，则：

* 顺序读取 1M 数据：
  + 内存：1.7 分钟
  + SSD：4.5 小时
  + 硬盘：11 天
* 硬盘寻址相当于在宿舍蹉跎一个月才搞清楚要追的妹子住在哪栋宿舍；
* 网络延时：
  + 同一数据中心来回要 6 天；
  + 加州到荷兰来回，相当于读完本科，并且又工作了半年；

现在能感受到 **内存**、**磁盘**、**网络** 之间速度的巨大差别了，粗略地讲：

* 硬盘比内存慢 1 千倍；
* 网络比内存慢 100 万倍，比硬盘慢 1 千倍；

注意网络延时与距离强相关，既可能非常快，也可能慢的要死。

Hadoop 通过将 **中间结果** 通过网络写入硬盘（分布式文件系统）实现容错，由于网络和硬盘的速度非常慢，所以 Hadoop 的容错代价很高。

而 Spark 把 **中间结果** 放在内存，通过纯函数对原始数据转换实现容错，并大大减少过程中的网络传输，因此速度更快，且提供远超 `map/reduce` 的接口，使用更加方便。

---

参考：

* [Big Data Analysis with Scala and Spark week 1](https://www.coursera.org/learn/scala-spark-big-data/lecture/D5o7O/latency)
