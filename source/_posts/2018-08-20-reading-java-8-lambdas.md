---
title: 读书 |《Java 8函数式编程》
date: 2018-08-20 22:33:47
tags:
  - Java
  - fp
categories: 读书
---

近年来函数式编程语言迎来了一次小爆发，热的发紫的 Spark 是用 Scala 实现的，Storm 是用 Coujure 实现的，Jetbrains 家的 Kotlin 也在紧锣密鼓的攻城略地，这些新型语言可利用庞大的 Java 生态系统，并且在语言特性上大大超越了保守的原住民 Java，程序员们当然快活，但 Java 却坐不住了，于是有了 Java 8 大刀阔斧的进步。

在我看来，Java 8 最重要的新特性包括：

* lambda 表达式（包括用 lambda 表达式增强的集合框架）
* `Optional`
* `Stream`
* `CompletableFuture`

说是新特性其实很勉强，因为 Java 8 早在 2014 年 3 月就发布了，而且 Java 9 在去年 9 月份也发布了，广大吃瓜群主肯定吃了一惊，哈哈。

回到本书，作为一本不到 150 页的小书，[Java 8函数式编程](https://book.douban.com/subject/26346017/) 对以上 4 点特性都有介绍，尤其 lambda 表达式和 `Stream` 接口，`Optional` 和 `CompletableFuture` 相对着墨不多，当然这并不意味它们不重要，相反，这两者相当常用，`Optional` 可以帮助消除臭名昭著的 NPE，`CompletableFuture` 则是语言内置的异步编程工具，且作为一个 monad，具备极强的可组合性，非常实用！

除语言特性外，第七章还介绍了如何调试、测试 lambda 表达式和 `Stream`，第八章介绍了新特性是如何影响程序设计的，包括对设计模式的影响、如何实现 DSL以及如何表达 SOLID 原则，这部分让我受益较大。

综合来看，全书篇幅虽短，但讲解全面、内容实用，非常值得一读。
