---
title: CIS 194 | Typeclass 和 IO
date: 2018-07-08 22:56:14
tags:
  - fp
  - CIS 194
  - Haskell
  - Type Class
categories: 技术
---

最近两周完成了 [CIS 194](http://www.seas.upenn.edu/~cis194/spring15/lectures.html) 第 4 章、第 5 章的学习，其中

* 第 4 章介绍 typeclass
* 第 5 章介绍 IO 相关内容

简单总结一下。

<!-- more -->

## Typeclass

Haskell 利用 typeclass 实现 ad-hoc 多态，一个 typeclass 是一组操作的集合（a set of operations），操作一般抽象为函数，因此 typeclass 是一组函数的集合（a set of functions）。

如果特定 type 支持 typeclass 定义的行为/操作/函数，则称该 type 是该 typeclass 的成员（member）或实例（instance）。

至于该 type 如何支持 typeclass 的行为/操作/函数，不同语言有不同的实现机制，比如：

* Scala 利用 `implicit` + 特质实现
* Haskell 利用 `instance` 关键字实现

Haskell 对 typeclass 的支持是语言内置的：

* `class` 定义 typeclass
* `instance` 定义 instances of typeclass
  + 为特定 type 实现 typeclass instance 后，该 type 就可以直接使用 typeclass 的函数

第 4 章课后作业即为多项式实现了 `Num` type class，所以多项式可以直接使用 `Num` 定义的 `+` 等函数：

```Haskell
(2*x^2 + x + 10) + (x^3 + x^2 + x)
```

就好像多项式本身定义了 `+` 函数一样，完全可以以假乱真。

Scala 语言本身没有显式支持 typeclass，而是通过 `implicit` 机制 + 特质 hack 出的 typeclass，而 `implicit` 和特质又不仅仅用于实现 typeclass，所以导致很难在代码中识别 typeclass（更多细节，请参考 {% post_link 2018-06-05-type-class-simulacrum 【译】Type class in Scala %}）。

## ADT

Haskell 中定义 ADT 非常方便：

```Haskell
infixr 5 :-:
data List' a = Empty | a :-: (List' a)
  deriving (Show, Read, Eq, Ord)

λ> 1 :-: 2 :-: Empty
1 :-: (2 :-: Empty)
```

* `List' a` 为 type constructor
* `Empty` 和 `:-:` 为 value constructor

作为对比，Scala 定义 ADT 就相对繁琐一些，若要定义 `:-:` 操作符则需要更多代码：

```Scala
sealed trait List[+A]

case class Cons[+A](head: A, tail: List[A]) extends List[A]
case object Nil                             extends List[Nothing]
```

Scala 用 `case` 类定义 ADT 时，同时生成了很多便利方法，Haskell 需要用 record 语法实现：

```Haskell
data Person = Person { firstName :: String
                     , lastName :: String
                     , age :: Int
                     , height :: Float
                     , phoneNumber :: String
                     , flavor :: String
                     } deriving (Show)
```

## IO

众所周知，Haskell 有三大特点：

* functional
* lazy
* purity

其中 purity 要求 Haskell 禁止存在副作用，进而导致其 IO 特性比较特殊，第 5 章练习题涉及很多 IO 函数。

如果需要了解更多实现细节，可以参考《Scala 函数式编程》第 13 章。

---

参考：

* [CIS 194 week 4 Typeclass homework](http://www.seas.upenn.edu/~cis194/spring15/hw/04-typeclasses.pdf)
* [CIS 194 week 5 IO homework](http://www.seas.upenn.edu/~cis194/spring15/hw/05-IO.pdf)