---
title:  Scala | case 语句究竟是什么类型？
date: 2018-06-07 21:55:21
tags:
  - Scala
  - case statement
  - 偏函数
categories: 技术
---

在 {% post_link scala-case-use-cases Scala | case 关键字的 4 种用法 %} 一文中我提到 case 语句可以创建偏函数字面值，实际上这种说法并不完全正确。

Scala 中 case 语句形如：

```Scala
{ case p1 => e1; case p2 => e2; ...; case pn => en }
```

实际上，仅给定 case 语句本身，我们只能确定它是一个 **匿名函数**，但 **无法** 确定该匿名函数的具体类型，根据 case 语句所处上下文对它的期待，它可以是：

* 普通函数（`FunctionN`）的匿名值
* 偏函数（`PartialFunction`）的匿名值

<!-- more -->

## 普通函数

`Seq.map` 的参数类型为 `A => B`（`Function1[A, B]`），因此若用 case 语句作为其实参，则 case 语句类型就是 `A => B`：

```Scala
// List(2, 4, 6)
Seq(1, 2, 3).map {
  case n ⇒ n * 2
}
```

该场景下 `{ case n => n * 2 }` 将被 Scala 编译器展开成 `Function1[Int, Int]` 实例：

```Scala
new Function[Int, Int] {
  override def apply(v1: Int) = v1 match {
    case n ⇒ n * 2
  }
}
```

## 偏函数

`Seq.collect` 的参数类型为 `PartialFunction[A, B]`，因此若用 case 语句作为其实参，则 case 语句类型为 `PartialFunction[A, B]`：

```Scala
// List(2, 4, 6)
Seq(1, 2, 3, "A").collect {
  case n: Int ⇒ n * 2
}
```

该场景下 `{ case n: Int => n * 2 }` 将被 Scala 编译器展开为 `PartialFunction[Int, Int]` 实例：

```Scala
new PartialFunction[Int, Int] {
  override def isDefinedAt(x: Int) = x match {
    case _: Int ⇒ true
    case _      ⇒ false
  }

  override def apply(v1: Int) = v1 match {
    case n: Int ⇒ n * 2
  }
}
```

---

参考：

* 知乎问题：[Scala 中的case关键字在这里是什么意思？](https://www.zhihu.com/question/29175392/answer/124176745)
* Cousera 公开课 [Functional Program Design in Scala - Week 1](https://www.coursera.org/learn/progfun2/lecture/zsnJ0/recap-functions-and-pattern-matching)