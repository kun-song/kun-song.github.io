---
title: Type Class | 基于 Monoid 和 Foldable 的 sum 函数
date: 2018-06-03 21:42:15
tags:
  - Scala
  - fp
  - Cats
  - Monoid
  - Foldable
  - Type Class
categories: 技术
---

在 {% post_link 2018-06-02-scala-3-polymorphism 基本概念 | 多态知多少？%} 一文中我说过特设多态在 Scala 中非常常见，且 Scala 利用 type class 实现特设多态。

>* type class 源于 Haskell
>* Scala 支持 type class 和函数重载两种方式实现特设多态

本文通过实现一个简洁、通用的 `sum` 函数，展示 type class 的强大之处。

<!-- more -->

## 乞丐版

最简单的 `sum` 是对 `List[Int]` 求和，可用 `fold` 实现：

```Scala
def sum(xs: List[Int]): Int = xs.foldLeft(0)(_ + _)
```

若要对 `List[String]` 求和，需重新实现一个 `sum`：

```Scala
def sum(list: List[String]): String = list.foldRight("")(_ ++ _)
```

若要对 `List[Set]` 求和呢，不幸的是，还得重新实现 `sum`：

```Scala
def unionSets[A](list: List[Set[A]]): Set[A] = list.foldRight(Set.empty[A])(_ union _)
```

## 升级版

重复是程序员的大敌，遇到重复代码时，需要考虑抽取不同点，将相同点抽象为通用函数，具体到 `sum` 函数：

* 相同点：都使用 `foldLeft` 实现
* 不同点：`foldLeft` 的两个参数不同

其实 `sum` 的不同点正好可以用 `Monoid` type class 表示，`Monoid` 由两部分组成：

* 二元操作 `combine`，类型为 `(A, A) => A`
* 单位元 `empty`，类型为 `A`

>更多关于 `Monoid` 的内容，参考我的 [Scala with Cats 笔记](https://github.com/satansk/scala-with-cats/blob/master/chapter02/Definition_of_a_Monoid.md)

可以很容易自定义一个 `Monoid`：

```Scala
trait Monoid[A] {
  def combine(x: A, y: A): A
  def empty: A
}
```

使用 `Monoid` 重构 `sum` 函数：

```Scala
def sum[A](xs: List[A])(m: Monoid[A]): A =
  xs.foldLeft(m.empty)(m.combine)
```

为方便使用，将 `Monoid` 定义为 `implicit` 参数：

```Scala
def sum[A](xs: List[A])(implicit m: Monoid[A]): A =
  xs.foldLeft(m.empty)(m.combine) 
```

这样就可以省略 `Monoid` 参数了：`sum(List(1, 2, 3))`。

Scala 专门为 type class 提供了 context bound 语法，用于简化 `implicit` 参数定义：

```Scala
def sum[A: Monoid](xs: List[A]): A = {
  val m = implicitly[Monoid[A]]
  xs.foldLeft(m.empty)(m.combine)
} 
```

* 通过 `implicitly` 函数获取隐式作用域中的值

只要隐式作用域中存在 `Monoid[A]` 实例，就可以通过 `sum` 对 `List[A]` 求和，`sum` 已经非常通用了，但还不够。

## 终极版

`sum` 目前只能对 `List` 求和，如果要对 `Option`、`Vector` 等求和怎么办呢，难道再重复实现一个 `sum`？

仔细观察 `sum` 的实现，除 `foldLeft` 函数外，它并不依赖 `List` 的其他属性，那能不能把 `foldLeft` 抽象出来呢？

当然可以！

`foldLeft` 并非 `List` 专有函数，Scala 世界中有无数类型具备 `foldLeft` 特性，因此完全可以把 `foldLeft` 抽象为 type class：

```Scala
trait FoldLeft[F[_]] {
  def foldLeft[A, B](xs: F[A], zero: B, f: (B, A) => B)
} 
```

`FoldLeft` 特质即为 type class，表示所有具备 `foldLeft` 函数的类型，其中 `F` 是一个类型构造器（type constructor）。

利用 `FoldLeft` 重构 `sum` 函数：

```Scala
def sum[A: Monoid, F[_]: FoldLeft](xs: F[A]): A = {
  val m = implicitly[Monoid[A]]
  val f = implicitly[FoldLeft[F]]
  f.foldLeft(xs, m.empty, m.combine)
}
```

现在 `sum` 没有任何写死的类型，`sum` 接受两个类型参数：`A` 和 `F[_]`，并通过两个 type class 限制必须存在 `Monoid[A]` 和 `FoldLeft[F]` 实例，并依赖 `FoldLeft` 和 `Monoid` 约束的行为实现 `sum` 函数。

>type class is a class/group of types，这组类型必须满足 type class 描述的行为，具体而言属于 `Monoid` 的类型必须具备 `combine` 和 `zero` 两个函数，且它们必须满足 `Monoid` 定律。

## 如何使用？

虽然我们已经拥有了近乎完美的 `sum` 函数，但要实际使用它还需提供对应的 `Monoid`、`FoldLeft` 实例，因为 type class 仅仅描述行为，实际实现由 type class instances 负责。

例如若要对 `List[Int]` 求和，需要提供 `Monoid[Int]` 和 `FoldLeft[List]` 实例：

```Scala
object Monoid {
  implicit val intMonoid: Monoid[Int] =
    new Monoid[Int] {
      override def combine(x: Int, y: Int): Int = x + y
      override def empty: Int = 0
    }
}

object FoldLeft {
  implicit val listFoldLeft: FoldLeft[List] =
    new FoldLeft[List] {
      override def foldLeft[A, B](xs: List[A], zero: B, f: (B, A) ⇒ B): B =
        xs.foldLeft(zero)(f)
    }
}
```

分别将它们导入当前作用域：

```Scala
import Monoid._
import FoldLeft._
```

之后即可对 `List[Int]` 求和：

```Scala
sum(List(1, 2, 3))  // 6
```

若要对 `Vector[String]` 求和，则需提供 `Monoid[String]` 和 `FoldLeft[Vector]` 实例！

等等，不是说好了消除重复代码吗，怎么还要写这么多 helper 代码？？？

好吧，我从 3 个方面为这些 helper 代码做下辩解：

首先，Scala 没有内置 type class 特性，我们是通过 `trait` + `implicit` 自己实现的 type class，完全从零开始，不可避免要写很多代码。

其次，作为一个库设计者（好吧，`sum` 也算一个微小的库），我们通过实现上的复杂，换取到清晰、简洁的接口，这是一种设计权衡。

最后，实际使用中，我们根本不需要、也不推荐自行实现 type class，一般使用 [Cats](https://typelevel.org/cats/)、[Scalaz](https://github.com/scalaz/scalaz) 等第三方库，它们包含常用类型的 type class instances 实现，直接导入即可使用。

## Cats 版

为了证明 type class 实际使用并不繁琐，我们用 Cats 重构 `sum`：

```Scala
import scala.language.higherKinds

import cats.Monoid
import cats.Foldable

def sum[A: Monoid, F[_]: Foldable](xs: F[A]): A = {
  val m = Monoid[A]
  val f = Foldable[F]
  f.foldLeft(xs, m.empty)(m.combine)
}
```

>Cats 中每个 type class 的伴生对象都有一个 `apply` 函数，效果与 Scala 内置的 `implicitly` 函数等同。

使用时，只需要 `import` 对应类型的 type class 实例即可，例如对 `List[Int]` 求和：

```Scala
import cats.instances.int._
import cats.instances.list._

sum(List(1, 2, 3))  // 6
```

对 `Vector[String]` 求和：

```Scala
import cats.instances.vector._
import cats.instances.string._

sum(Vector("A", "B", "C"))  // ABC
```

## 总结

type class 源于 Haskell，最初为实现特设多态而引入，Scala 没有内置该特性，但允许用户通过 `trait` 和 `implicit` 参数自行实现。

[Scala 函数式编程](https://book.douban.com/subject/26772149/) 讲解了很多 type class 的实现细节，它们很多都可以在 Cats/Scalaz 中找到。

type class 属于 Scala 中比较偏 FP 的特性，若非 FP 粉丝，实际项目中一般很少使用。

---

参考：

* [Cats 官方文档](https://typelevel.org/cats/typeclasses.html)
* [Herding Cats](http://eed3si9n.com/herding-cats/sum-function.html)
