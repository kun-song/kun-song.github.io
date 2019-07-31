---
title: 牛顿法求平方根（Scala 版）
date: 2018-06-10 21:52:34
tags:
categories: Scala
---

牛顿法求平方根是 [计算机程序的构造与解释](https://book.douban.com/subject/1148282/) 一书中的经典例子。

算法如下：

若要计算 `sqrt(x)`：

1. 以一个初始猜测 `y` 开始（例如 y = 1）
2. 重复求 `y` 和 `x/y` 的均值，逼近真实的平方根

<!-- more -->

每次迭代可以表示为：

```Scala
def sqrtIter(guess: Double, x: Double): Double =
  if (isGoodEnough(guess, x)) guess
  else sqrtIter(improve(guess, x))
```

其中 `isGoodEnough` 和 `improve` 很容易定义：

```Scala
def isGoodEnough(guess: Double, x: Double) =
  Math.abs(x - square(guess)) < 0.001

def improve(guess: Double, x: Double) =
  (guess + x / guess) / 2

def square(n: Double) = n * n
```

而 `sqrt` 函数可借助 `sqrtIter` 实现：

```Scala
def sqrt(x: Double): Double = sqrtIter(1, x)
```

最后代码为：

```Scala
def sqrt(x: Double): Double = sqrtIter(1, x)

def sqrtIter(guess: Double, x: Double): Double =
  if (isGoodEnough(guess, x)) guess
  else sqrtIter(improve(guess, x), x)

def isGoodEnough(guess: Double, x: Double) =
  Math.abs(x - square(guess)) < 0.001

def improve(guess: Double, x: Double) =
  (guess + x / guess) / 2

def square(n: Double) = n * n
```

因为 `sqrtIter`、`isGoodEnough`、`improve` 和 `square` 都是 `sqrt` 的辅助函数，因此可以定义在 `sqrt` 函数内部：

```Scala
def sqrt(x: Double): Double = {
  @tailrec
  def sqrtIter(guess: Double): Double =
    if (isGoodEnough(guess, x)) guess
    else sqrtIter(improve(guess, x))

  def isGoodEnough(guess: Double, x: Double) =
    Math.abs(x - square(guess)) < 0.001

  def improve(guess: Double, x: Double) =
    (guess + x / guess) / 2

  def square(n: Double) = n * n

  sqrtIter(1)
}
```

又因为 `sqrt` 内部所有函数都可以访问参数 `x`，因此内部函数可以去掉 `x` 参数（闭包）：

```Scala
def sqrt(x: Double): Double = {
  @tailrec
  def sqrtIter(guess: Double): Double =
    if (isGoodEnough(guess)) guess
    else sqrtIter(improve(guess))

  def isGoodEnough(guess: Double) =
    Math.abs(x - square(guess)) / x < 0.001

  def improve(guess: Double) = average(guess, x / guess)

  def square(n: Double) = n * n

  def average(x: Double, y: Double) = (x + y) / 2

  sqrtIter(1)
}
```

但我们的 `sqrt` 有两个问题：

* 对于非常小的数值，例如 `sqrt(1e-10)` 求值结果为 0.03125000106562499，明显错误
* 对于非常大的数值，例如 `sqrt(1e100)`，计算根本无法停止

原因在于 `isGoodEnough` 实现存在 bug：

```Scala
def isGoodEnough(guess: Double, x: Double) =
  Math.abs(x - square(guess)) < 0.001
```

改为：

```Scala
def isGoodEnough(guess: Double, x: Double) =
  Math.abs(x - square(guess)) / x < 0.001
```

即可。
