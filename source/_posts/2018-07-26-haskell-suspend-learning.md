---
title: 暂时出坑 Haskell
date: 2018-07-26 21:26:10
tags:
  - Haskell
  - CIS 194
categories: 公开课
---

最近一个多月（见 {% post_link 2018-06-15-haskell-begin-cis-194 正式入坑 Haskell %}）的业余时间基本都在学习 Haskell，学习材料是：

* 宾夕法尼亚大学的 [CIS 194](http://www.seas.upenn.edu/~cis194/spring15/lectures.html)
* [Learn You a Haskell for Great Good!](http://learnyouahaskell.com/chapters)

其中 CIS 194 完成了第一周到第六周的学习内容，而《Haskell 趣学指南》则基本读完，笔记、代码都在 github 上：

* [CIS 194 笔记](https://github.com/satansk/cis-194)
* [Haskell 趣学指南笔记](https://github.com/satansk/learn-you-a-haskell)

<!-- more -->

前面博客中说过，选择 Haskell 的原因是：

* 网上关于 FP 的资料、博客大部分以 Haskell 为例
* Haskell 社区很活跃
* Haskell 可以加持 Scala

通过这段时间的学习，掌握了 Haskell 的基本特性，其中递归、高阶函数等与 Scala 很类似，不再多表，不那么类似的特性有：

* 极其简洁的 ADTs
* typeclass 支持（`class` + `instance` 关键字）

Scala 的 ADTs 通过 case 类实现，比较繁琐，而 Haskell 通过 `data` 实现 ADTs 异常简洁，很优美。

Haskell 中 typeclass 使用非常普遍，大量的 typeclass 例子让我对 typeclass 理解大大加深，也领略到了 Haskell 简洁、纯粹之美，以前用 Scala 实现 typeclass 时总是云里雾里，这下追本溯源，终于豁然开朗：

* Haskell：`class` 定义 typeclass，`instance` 实现 typeclass instance；
* Scala：`trait` 定义 typeclass，对象 + `implicit` 实现 typeclass instance；

虽然 Haskell 还有很多很多高阶特性，但 Haskell 的特性是学不完的，只能需要根据随时学习，不能一蹴而就。目前我对 Haskell 的掌握已经基本达到当初的目的，所以虽然 Haskell 公开课没有学完，但我认为可以暂告一段落，毕竟学习与实践不能偏废，接下来需要实践啦。

接下来可以学习一点 PL 实现，比如：

* [Essentials of Programming Languages](https://book.douban.com/subject/3136252/)

然后可以写写解释器，比如：

* [Write Yourself a Scheme in 48 Hours](https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours)

这两个任务都可以用 Haskell 和 Scala 实现，是很好的练手项目，有空一定要做做。
