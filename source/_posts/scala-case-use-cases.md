---
title: Scala | case 关键字的 4 种用法
tags: Scala
categories: 技术
abbrlink: 38736
date: 2018-05-15 22:44:37
---

在 Scala 中 `case` 关键字主要有 4 种使用场景，分别是：

<!-- more -->

## 模式匹配

```scala
10 match {
  case n if n % 2 == 0 ⇒ "even"
  case n if n % 2 != 1 ⇒ "odd"
}
```

## `case` 类

```Scala
case class Number(value: Int)
```

## `try-catch`

```Scala
def toInt(s: String): Int =
  try {
    s.toInt
  } catch {
    case _: Throwable ⇒ 0
  }
```

## 偏函数字面值

`case` 关键字可用于创建 `PartialFunction` 类型的函数字面值（匿名函数）：

```Scala
val xs = List(1, 2, 3, 4)

xs.filter {
  case n ⇒ n % 2 == 0
}
```

事实上，`case` 关键字是 **最常用** 的创建 `PartialFunction` 的方式，若不用 `case`，则只能手动创建 `PartialFunction` 实例：

```Scala
val fraction = new PartialFunction[Int, Int] {
  def apply(d: Int) = 42 / d
  def isDefinedAt(d: Int) = d != 0
}
```

使用 `case` 明显更加紧凑：

```Scala
val fraction: PartialFunction[Int, Int] = {
  case d: Int if d != 0 ⇒ 42 / d
}
```

注意，必须手动指定 `fraction` 的类型为 `PartialFunction`，否则 Scala 编译器会报错：

```Scala
val fraction = {
  case d: Int if d != 0 ⇒ 42 / d
}
```

也可以将 `fraction` 类型指定为普通函数，但这样就失去了偏函数的魔力：

```Scala
val fraction: Function[Int, Int] = {
  case d: Int if d != 0 ⇒ 42 / d
}
```