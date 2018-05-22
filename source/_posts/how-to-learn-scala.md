---
title: 我的 Scala 学习之路
tags: Scala
categories: 随笔
abbrlink: 41898
date: 2018-05-09 23:12:42
---

本文是我对过去一年学习 Scala 的总结，既是总结经验教训，也希望对初学者有所帮助。

<!-- more -->

## 初次接触

说来有趣，大约一年以前，我转到一个新的项目组，晨会上了解到有同事正在调研 Scala，每天该同事都汇报调研进度（虽然一般是还没开始、下周开始云云），我那时候刚转正，想着既然以后可能用到，那我得提前学习，毕竟机遇总是留给有准备的人，于是我开始在业余时间学习 Scala。

## 难以入门

当时在知乎搜索了很多如何学习 Scala 的文章，看到很多人推荐《Scala 函数式编程》（著名的小红书），我当然知道这书很难，但当时蜜汁自信，认为看书就要看最经典、最难的，虽然进度慢，但收获也大不是吗？

开始看书以后，我发现除了第一章读起来比较顺畅，从第二章开始就很吃力了。苦苦挣扎了几个周，加上上半年加班非常多，导致学习效果很不好。

而此时同事对 Scala 的调研早已结束，连个结论都没给出，Scala 毕竟偏向后端，让前端同事去调研，结果可想而知。

这段时间的学习让我熟悉了很多函数式编程的术语，自己对函数式编程也越来越好奇，遗憾的是 Scala 始终没有入门，而且 Scala 给我留下了难以学习的印象。

## 初窥门径

时间一晃到了去年的 7 月份，这时项目加班终于少了一些，而我按捺不住对 Scala 的好奇心，又一次尝试学习，这次我决定从知乎神课 [Programming Languanges](https://www.coursera.org/learn/programming-languages/home/welcome) 入门，关于该课程的评价建议可参考很多知乎回答，一句话总结：函数式编程入门不二之选，对学习 Scala 也大有裨益。

不得不说，这门课真的是物超所值，我那段时间只跟了 Part A，使用 Standard ML 讲述函数式编程的基本概念，例如：

* list 组合子（map, filter, fold 等）
* first-class function, high order function, closure, currying, partially applied function ...

这些概念今天来看非常基础，但对毫无函数式编程基础的同学来说却并不容易，非常建议大家学习下这门课，对于入门 Scala 非常有用，因为 Scala 的函数式编程与 Standard ML 非常类似（当然与 Haskell 更类似），公开课最大的优点是有练习题目，有实际编程机会，这对初学者帮助很大。

后来我又跟了 Scala 之父 Martin Odersky 的一门课：[Functional Programming Principles in Scala](https://www.coursera.org/learn/progfun1/home/welcome)，这门课主要讲述 Scala 的函数式编程，对面向对象几乎只字未提，习题稍微有点难度。

实事求是的讲，这门课比 Programming Languages 还是略逊一筹，课程内容不是很连贯，我认为最好先学下 Programming Languages Part A，然后在跟这门课，事半功倍。

跟完这两门公开课我对 Scala，或更广义的函数式编程基本入门了，接下来需要实践。

## 渐入佳境

前面说过，项目组对 Scala 的调研虎头蛇尾，不了了之，公司里没有实践机会了。那时的我手握锤子，到处找钉子，尝试过写自己的业余项目，但苦于没有好点子，只能作罢。

没有实际项目做那就继续学习呗，于是我开始重新学习 《Scala 函数式编程》。

有了公开课打下的基础，这本书也没有那么难理解了，我一口气读完了 1-6 章，这几章都是基础，主要讲了函数式数据结构（list/tree）、函数式 error-handling、stream、纯函数式状态等，其中很多内容在公开课中已经接触过，例如：

* list 在 Programming Languages Part A 中讲过
* stream 则在 Programming Languages Par B 中讲过
* 纯函数式状态在 Martin Odersky 的课中讲过

公开课的优点是容易入门，缺点是不够深入，小红书完美补足了公开课的缺点，足够深入，而且最赞的是小红书有大量的练习题目，引导读者实现了很多 Scala 标准库以及 Cats 中的数据结构，非常过瘾。

>我把小红书的练习代码放在了 [github](https://github.com/satansk/learning-fpinscala) 上，大家可以参考下。

学习过程中我得到一个教训：如果某本书、某个知识点你完全不懂，那么你肯定缺少相关基础，此时不要继续死磕，应该换个思路，去补下基础，基础补完后再重新回来。

读到第 7 章的时候我又几乎不懂了，哈哈，但有了前面的教训，我没有继续死磕第 7 章，我明白应该暂时放下书，在合适的时候再重新开始。

我那时认为自己已经有足够的 Scala 知识，是时候做点真实的项目了，我决定参与开源社区。

其实早在学校的时候，我就想参与开源，但一直苦于没有机会，于是暗下决心，这次一定要破门而入。我在知乎搜索了很多 Scala 相关的问题，不夸张的说，95% 的相关内容我都看过，我总结了 Scala 的主要项目有：

* Spark/Flink 等大数据相关应用；
* Akka/Akka HTTP
* Twitter 开源的很多框架
* Cats/Scalaz
* Kafka

最后我选择了 Akka/Akka HTTP，因为它们主要涉及分布式应用开发，而且有很多先进理念（例如反应式编程），更重要的是社区非常活跃，对新人非常友好。

要贡献代码，首先要阅读 CONTRIBUTING.md，巧的是阅读过程中就发现了一个小问题，赶紧提 PR，Akka 团队响应非常快，马上就合入了。

这个小小的 PR 给了我很大鼓励，原来开源社区并非高不可攀，于是周末在家也有事情干了，从 2017 9 2 日到今天，Akka/Akka HTTP 一共接受了我 19 个PR：

<img src="/images/akka-pr-20180514.png" alt="Akka PR 统计" style="width: 400px;"/>

<img src="/images/akka-http-pr-20180514.png" alt="Akka HTTP PR 统计" style="width: 400px;"/>

虽然贡献的代码不多，但收获却不少，实践出真知的确并非虚言，有高手 review 你的代码，确实成长的更快。

期间我把小红书第 7-12 章读完了，中间还读了两本书，是 typelevel 的 [Essential Scala](https://underscore.io/books/essential-scala/) 和 [Scala with Cats](https://underscore.io/books/scala-with-cats/)，读完这些书后，我对 type class 有了基本理解。

我也发现实际项目中 Cats/Scalaz 使用确实不多，毕竟并非每个人都是 Scala 函数式编程熟手，芸芸众生才是主体。当然如果组内同事都是 fp 粉，用下也未尝不可。

Scala 是一门很难精通的语言，知识点很多，但按我的理解，要享受 Scala 带来的方便之处，并不需要精通语言的方方面面，而是需要取舍，选择最合适的语言特性，至于如何取舍则需要高手把关。

## 展望未来

到目前为止，我对 Scala 也只是堪堪入门，但已经足够我写一些有趣的项目了，总体来说，学习 Scala 是很开心的，写 Scala 代码也是很开心的，希望未来能继续坚持下去，至于能不能用 Scala 吃饭并不重要。

## 总结

总结一些对 Scala 学习有用的资料：

* 公开课
  + [Programming Languanges](https://www.coursera.org/learn/programming-languages/home/welcome)
  + [Functional Programming Principles in Scala](https://www.coursera.org/learn/progfun1/home/welcome)
* 书
  + [Programming in Scala, Third Edition](https://book.douban.com/subject/26790779/)
  + [Scala 函数式编程](https://book.douban.com/subject/26772149/)
  + [Essential Scala](https://underscore.io/books/essential-scala/)
  + [Scala with Cats](https://underscore.io/books/scala-with-cats/)
* 网络资料
  + [Twitter Scala School](https://twitter.github.io/scala_school/)
  + [王宏江博客](http://hongjiang.info/)

前面没有提 Programming in Scala 和 Twitter Scala School，并非它们不够好，而是因为每个 Scala 学习者都应该读过它们 :)
