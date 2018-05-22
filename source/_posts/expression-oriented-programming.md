---
title: 面向表达式编程
tags:
  - fp
categories: 技术
abbrlink: 43893
date: 2018-05-15 08:35:13
---

维基百科有对 [面向表达式编程语言](https://en.wikipedia.org/wiki/Expression-oriented_programming_language) 的介绍，而面向表达式编程（expression oriented programming）也由此而来。

EOP 语言褒贬不一，我们不做评价，而了解面向表达式编程对理解函数式编程大有裨益，因为 **所有** 函数式语言都是 **面向表达式** 的。

<!-- more -->

## 语句 vs 表达式

语句（statement）、表达式（expression）是编程语言中常见的概念，两者区别如下：

* 语句
  1. 无返回值
  2. 仅使用其 **副作用**
* 表达式
  1. 必有返回值
  2. 一般无副作用

在 OOP 语言中，常常见到如下代码：

```Java
order.calculateTaxes()
order.updatePrices()
```

* 典型的 **语句**，因为没有使用它们的返回值（它们返回值毫无意义），仅通过其副作用来影响程序。

相对而言，EOP 风格的代码如下：

```Java
val tax = calculateTax(order)
val price = calculatePrice(order)
```

* 仅使用返回值，无副作用

使用纯函数式编程语言时，我们会写很多表达式，用表达式组成纯函数，这既是 FP 风格（函数式编程风格），又是 EOP 风格（面向表达式风格）。

## Scala 是面向表达式的

作为函数式编程语言，Scala 推崇使用面向表达式编程，时刻将 EOP 记在心里，有助于写出地道的 Scala 代码。

`if` 语句是表达式：

```Scala
val max = if (a > b) a else b
```

模式匹配也是表达式：

```Scala
val evenOrOdd = i match {
    case i if i % 2 == 1  => "odd"
    case i if i % 2 == 0  => "even"
}
```

try-catch 也是表达式：

```Scala
def toInt(s: String): Int = {
    try {
        s.toInt
    } catch {
        case _ : Throwable => 0
    }
}
```

总之一句话，Scala 中几乎所有东西都是表达式。