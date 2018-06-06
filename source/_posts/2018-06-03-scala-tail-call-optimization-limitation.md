---
title: Scala | 不完美的尾递归优化
date: 2018-06-03 15:50:26
tags:
  - Scala
  - fp
  - 尾递归
  - 蹦床
categories: 技术
---

前几天我在 {% post_link 2018-05-16-tail-recursion 为什么需要尾递归？%} 一文中介绍了尾递归的基本概念，以及 Scala 编译器对尾调用（递归）的优化。

不幸的是，由于 JVM 实现的局限性（参考知乎问题 [为什么 JVM 下的语言尾递归优化似乎都不完美？](https://www.zhihu.com/question/22627957)），Scala 对尾调用的优化 **非常有限**：

* 只能优化直接尾调用（self-tail calls）

该缺陷导致 Scala 编译器无法优化 mutual recursion，大大降低了递归的实用性。

<!-- more -->

>* mutual recursion 在 FP 中非常常见，在某些领域中 mutual recursion 是非常自然的解法（例如递归下降解析器）
>* Haskell 并无该缺陷，可以优化 mutual recursion

## 互递归实例

首先看一个 mutual recursion 的例子：

```Scala
def isEven(n: Int): Boolean =
  if (n == 0) true
  else isOdd(n - 1)

def isOdd(n: Int): Boolean =
  if (n == 0) false
  else isEven(n - 1)
```

`isEven` 和 `isOdd` 互相调用，组成 mutual recursion，Scala 编译器无法对其进行优化，`isEven(100000)` 即可造成栈溢出。

## 蹦床：优化互递归

蹦床（trampoline call）可以避免 mutual recursion 栈溢出，但要承受一定的性能损耗，使用 `TailRec` 重写上面的例子：

```Scala
import scala.util.control.TailCalls.TailRec
import scala.util.control.TailCalls._

def isEven(n: Int): TailRec[Boolean] =
  if (n == 0) done(true)
  else tailcall(isOdd(n - 1))

def isOdd(n: Int): TailRec[Boolean] =
  if (n == 0) done(false)
  else tailcall(isEven(n - 1))
```

此时 `isEven(1000000000).result` 都不会发生栈溢出了，可以放心使用。

## 蹦床：优化普通递归

除尾递归外，普通递归也可用 `TailRec` 优化，例如：

```Scala
def  fib(n: Int): Int =
  if (n < 2) n
  else fib(n - 1) + fib(n - 2)
```

* 该解法既不是尾递归，也不是互递归

用蹦床优化 `fib` 后为：

```Scala
def  fib(n: Int): TailRec[Int] =
  if (n < 2) done(n)
  else for {
    x ← tailcall(fib(n - 1))
    y ← tailcall(fib(n - 2))
  } yield x + y
```

再也不会出现栈溢出了！

## 蹦床原理

蹦床原理非常简单，即利用 thunk 实现延迟计算（更多关于延迟计算的内容，请参考 {% post_link 2018-05-22-racket-lazy-evaluation Racket | 拨开延迟计算的迷雾 %}），一个非常简单的 `TailRec` 实现如下：

```Scala
sealed trait TailRec[A]
case class Call[A](thunk: () ⇒ TailRec[A]) extends TailRec[A]
case class Done[A](result: A)              extends TailRec[A]
```

短短的 3 行代码就能解决 `isEven` 和 `isOdd` 互递归的栈溢出问题：

```Scala
def isEven(n: Int): TailRec[Boolean] =
  if (n == 0) Done(true)
  else Call(() ⇒ isOdd(n - 1))

def isOdd(n: Int): TailRec[Boolean] =
  if (n == 0) Done(false)
  else Call(() ⇒ isEven(n - 1))

trampoline(isEven(10000000))
```

`fib` 实现中使用了 for 表达式，for 表达式依赖 `map`、`flatMap` 等函数，为 `TailRec` 添加这些函数后即可将 `TailRec` 用于 for 表达式，该任务留待以后有时间完成吧。

## 总结

Scala 选择 JVM 作为目标平台，既享受了 JVM 宇宙第一生态的优势，也得忍受 JVM 的种种不足，尾递归优化即为其中一个例子。

当遇到 mutual recursion 时，要么改为迭代实现，要么使用蹦床，若用蹦床，需要确定可以忍受蹦床带来的性能损失。

---

参考：

* [Tail calls, @tailrec and trampolines](http://blog.richdougherty.com/2009/04/tail-calls-tailrec-and-trampolines.html)
* [为什么 JVM 下的语言尾递归优化似乎都不完美？](https://www.zhihu.com/question/22627957)
* 维基百科 [Mutual recursion](https://en.wikipedia.org/wiki/Mutual_recursion)
* [DO YOU LIKE SCALA? GIVE HASKELL A TRY!](https://www.fpcomplete.com/blog/2016/11/comparison-scala-and-haskell)