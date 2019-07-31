---
title: 读书 | Scala with Cats
date: 2018-08-06 23:03:00
tags:
  - Cats
categories: 读书
---

看了小红书（Functional Programming in Scala）后，脑子里充斥了各种函数的实现，但这些 Monad 啊，Functor 啊到底有什么用呢？实际项目中怎么把这些高大上的概念用上呢？

实际项目中当然不能手动实现各种 typeclass，因为 Cats 已经为我们提供了非常多有用的 type class 了（还有 Scalaz，二者很像）。

<!-- more -->

Scala with Cats 是 typelevel 提供的众多 Scala 电子书之一，免费可得，提供 html、epub、pdf 等多种格式，当然这都不是关键，关键是这本书写的非常好，我这么说当然没有说服力，大家可以看下 goodreads 上的评价，点我。

这本书不但教会你函数式编程的基本理论，比如 `Monoid`, `Semigroup`, `Functor`, `Monad`, `Applicative` 等等，还会告诉你 Cats 是如何实现这些 typeclass 的，更赞的是每章都提供了习题，用于讲解怎么把 Cats 用于实际项目。

看到这里会发现，Scala with Cats 与 Functional Programming in Scala 是有部分重合的，其中函数式理论两者都有涉及，typeclass 实现其实也有部分重合，因为 Cats 本身实现了很多 fpinscala 中的 typeclass。

Scala with Cats 比小红书更贴近实战，更让你马上将 typeclass 用于实际项目。

另外，本书的读书笔记放在 github 上了，包含本书的主要内容，以及习题思路解答等等，点我查看。
