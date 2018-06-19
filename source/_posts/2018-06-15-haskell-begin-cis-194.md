---
title: CIS 194 | 正式入坑 Haskell
date: 2018-06-15 08:35:45
tags:
  - Haskell
  - 公开课
  - fp
  - CIS 194
categories: 技术
---

Martin Odersky 曾经说过：

>Scala is a gateway drug to Haskell.

虽是戏言，却一语成谶。

虽然 Scala 是一门不错的 FP 语言，但由于兼顾 OO，所以很多 FP 概念在 Scala 中显得不够清晰，即语法噪声很多。

我对真正的 FP 语言越来越好奇，想深入了解一门，以便破除 Scala 中的一些迷惑之处，这时我有几个选择：

* Haskell
* Standard ML
* Racket/Scheme

<!-- more -->

后面两个很早就接触了。

首先 Racket/Scheme 语法简单，又有 SICP 加持，应该是很好的选择，但它有个致命“缺点”：动态语言，好吧，我知道动态语言很灵活，写起来很爽，但目前来说真的提不起兴趣 :(

我个人非常喜欢 Standard ML，正统的函数式编程语言，[Purely Functional Data Structures](https://book.douban.com/subject/1755557/) 的例子代码就是 Standard ML，它的缺点是，相关资料比较老，目前用的人比较少，网上的 FP 博客很少用 Standard ML，sad...

剩下的就只有 Haskell 了，虽然 Haskell 过于简洁的语法让我不太适应，但它有如下优点：

* 网上关于 FP 的资料、博客大部分以 Haskell 为例
* Haskell 社区很活跃
* Haskell 可以加持 Scala

虽然 Standard ML 也可以加持 Scala，但它没有 type class，而 Scala 的 type class 几乎就是 Haskell 的翻版。

于是我决定入坑 Haskell 了。

计划从宾大的 [CIS 194](http://www.seas.upenn.edu/~cis194/fall16/index.html) 开始，这门课资料齐全，完全可以自学，网上评价也不错，恩，就从它开始吧。
