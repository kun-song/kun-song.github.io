---
title: Scala | 一篇文章读懂高阶函数
date: 2018-06-12 21:02:39
tags:
  - 高阶函数
  - 柯里化
  - 部分应用函数
  - 柯里化
categories: Scala
---

首先，高阶函数是一种特殊的函数：

* 接受函数参数，or
* 返回函数

高阶函数非常便于 **组合函数**。

本文通过逐渐优化一个求和函数，展示高阶函数、匿名函数、柯里化、部分应用等多种函数式编程概念。

<!-- more -->

## 引子

实现一个计算 [from, to] 区间内整数之和的函数，这非常简单：

```Scala
def sumInts(from: Int, to: Int): Int =
  if (from > to) 0
  else from + sumInts(from + 1, to)
```

求 [from, to] 之间整数的立方和呢？也很简单：

```Scala
def cube(n: Int): Int = n * n * n

def sumCubes(from: Int, to: Int): Int =
  if (from > to) 0
  else cube(from) + sumCubes(from + 1, to)
```

求 [from, to] 之间整数的阶乘之和呢？代码与 `sumInts` 和 `sumCubes` 非常类似：

```Scala
def fact(n: Int): Int = if (n == 0) 1 else n * fact(n - 1)

def sumFacts(from: Int, to: Int): Int =
  if (from > to) 0
  else fact(from) + sumFacts(from + 1, to)
```

`sumInts`、`sumCubes` 和 `sumFacts` 主体代码非常类似，能不能抽象出公共部分呢？

## 第一次优化：高阶函数

将 `sumInts`、`sumCubes` 和 `sumFacts` 的差异点抽象为函数，并作为参数传入：

```Scala
def sum(f: Int ⇒ Int, from: Int, to: Int): Int =
  if (from > to) 0
  else f(from) + sum(f, from + 1, to)
```

`sum` 接受一个类型为 `Int => Int` 的函数作为参数，因此是高阶函数。

通过 `sum` 可以重新实现 `sumInts`、`sumCubes` 和 `sumFacts`：

```Scala
def cube(n: Int): Int = n * n * n
def fact(n: Int): Int = if (n == 0) 1 else n * fact(n - 1)

def sumInts(from: Int, to: Int): Int = sum(identity, from, to)
def sumCubes(from: Int, to: Int): Int = sum(cube, from, to)
def sumFacts(from: Int, to: Int): Int = sum(fact, from, to)
```

* 其中 `identity` 类似 `x => x` 这样的函数。

**求和逻辑** 被抽象在 `sum` 函数中，该逻辑在 `sumInts`、`sumCubes` 和 `sumFacts` 中获得复用。

美中不足的是，`cube`、`fact` 只是辅助函数，用完即扔，现在还要费神单独定义它们。

## 第二次优化：函数字面值

实际上，高阶函数的存在不可避免导致定义大量的临时辅助函数，它们存在的 **唯一作用** 就是作为高阶函数的参数，因此一般 FP 语言都会提供函数字面值（匿名函数），以简化高阶函数的使用。

>* 类似 100 是 `Int` 类型的字面值，(x: Int) => x + 1 是函数类型 `Int => Int` 的字面值；
>* 函数字面值仅仅是函数定义的语法糖；

利用函数字面值，可以简化 `sumInts` 等的定义，省去单独定义辅助函数：

```Scala
def sumInts(from: Int, to: Int): Int = sum(x ⇒ x, from, to)
def sumCubes(from: Int, to: Int): Int = sum(x ⇒ x * x * x, from, to)
```

而 `sumFacts` 的参数 `fact` 函数是递归函数，用匿名函数实现比较复杂，因此保持原状。

## 第三次优化：尾递归

目前 `sum` 函数实现为线性递归，当递归层次过深时，很容易导致栈溢出，这会严重影响 `sumInts` 等函数的可用性，因此将其改写为尾递归：

```Scala
import scala.annotation.tailrec

def sum(f: Int ⇒ Int, from: Int, to: Int): Int = {
  @tailrec
  def aux(acc: Int, from: Int): Int =
    if (from > to) acc
    else aux(acc + f(from), from + 1)

  aux(0, from)
}
```

现在 `sumInts`、`sumCubes` 等不会出现栈溢出了。

## 第四次优化：柯里化

通过把求和逻辑抽象到 `sum` 中，`sumInts` 等函数定义已经很简洁了：

```Scala
def sumInts(from: Int, to: Int): Int = sum(x ⇒ x, from, to)
def sumCubes(from: Int, to: Int): Int = sum(x ⇒ x * x * x, from, to)
def sumFacts(from: Int, to: Int): Int = sum(fact, from, to)
```

目前唯一的槽点是 `sumInts` 的参数 `from` 和 `to` 直接被传递给了 `sum`，毫无存在感，能把它们简化掉吗？

Yes, we can!

### 1. 柯里化函数定义：手动实现

重新定义 `sum`，新 `sum` 接受 `Int => Int` 函数作为参数，并返回 `(Int, Int) => Int` 函数：

```Scala
def sum(f: Int ⇒ Int): (Int, Int) ⇒ Int = {
  def sumF(from: Int, to: Int): Int =
    if (from > to) 0
    else f(from) + sumF(from + 1, to)

  sumF
}
```

* 我们在 `sum` 内部定义了一个 `(Int, Int) => Int` 函数，并将其作为 `sum` 的返回值
* `sumF` 是典型的闭包，引用外部变量 `f`

借助新 `sum`，我们重新定义 `sumInts` 等：

```Scala
def sumInts = sum(x ⇒ x)
def sumCubes = sum(x ⇒ x * x * x)
def sumFacts = sum(fact)
```

是不是更简洁了，哈哈！

`sum(x => x)` 计算结果为函数，可以直接调用：

```Scala
sum(x => x)(0, 10) == sumInts(0, 10)
```

* 从本例可看出，函数应用为左结合，即 `sum(x => x)(0, 10)` 为 `(sum(x => x))(0, 10)`

### 2. 柯里化函数定义：多参列表

高阶函数经常返回函数，因此 Scala 提供了响应的语法糖：

```Scala
def sum(f: Int ⇒ Int)(from: Int, to: Int): Int =
  if (from > to) 0
  else f(from) + sum(f)(from + 1, to)
```

通过定义时的多参列表，Scala 自动将其转换为柯里化函数，与手动返回 `sumF` 几乎完全相同，但用它定义 `sumInts` 时：

```Scala
def sumInts = sum(x ⇒ x)
def sumCubes = sum(x ⇒ x * x * x)
def sumFacts = sum(fact)
```

将抛出异常：

```Scala
Error:(8, 19) missing argument list for method sum in class A$A582
Unapplied methods are only converted to functions when a function type is expected.
You can make this conversion explicit by writing `sum _` or `sum(_)(_,_)` instead of `sum`.
def sumInts = sum(x ⇒ x)
                 ^
```

根据提示，`sum` 并非函数，而是方法，只有编译器确定需要函数的地方，才会将方法转换为函数，这里 `sum(x => x)` 处于 `sumInts` 方法体中，编译器无法确定此处需要一个函数，因为 `sum(x => x)` 无论返回任何类型 `sumInts` 定义都是合法的。

有两种解决方式，第一显式指定 `sumInts` 的返回类型，编译器将自动将 method 转换为 function：

```Scala
def sumInts: (Int, Int) ⇒ Int = sum(x ⇒ x)
def sumCubes: (Int, Int) ⇒ Int  = sum(x ⇒ x * x * x)
def sumFacts: (Int, Int) ⇒ Int  = sum(fact)
```

第二，根据异常提示，显式使用 `_` 强制将 method 转换为 function：

```Scala
def sumInts = sum(x ⇒ x) _
def sumCubes = sum(x ⇒ x * x * x) _
def sumFacts = sum(fact) _
```

#### 一点理论

多参函数原理很简单，多参函数定义：

```Scala
def f(args1)...(argsn) = E
```

等价于：

```Scala
def f(args1)...(argsn-1) = {
  def g(argsn) = E
  g
}
```

或更简洁的：

```Scala
def f(args1)...(argsn-1) = argsn => E
```

对剩余的 `(args1)...(argsn-1)` 重复应用该原理，最后结果为：

```Scala
def f = (args1 => (args2 ... => (argsn => E)))
```

函数类型是 **右结合**，因此简化为：

```Scala
def f = args1 => args2 ... => argsn => E
```

`f` 没有任何参数了！

## 引子 2

实现一个对 [from, to] 区间整数求乘积的函数，加持所有从 `sum` 获取的经验，将其定义为：

```Scala
def product(f: Int ⇒ Int)(from: Int, to: Int): Int =
  if (from > to) 1
  else f(from) * product(f)(from + 1, to)
```

借助 `product` 很容易定义 `fact`：

```Scala
def fact(n: Int): Int = product(x ⇒ x)(1, n)
```

## 第五次优化：抽象 sum 和 product

看一下 `sum` 和 `product` 的定义：

```Scala
def sum(f: Int ⇒ Int)(from: Int, to: Int): Int =
  if (from > to) 0
  else f(from) + sum(f)(from + 1, to)

def product(f: Int ⇒ Int)(from: Int, to: Int): Int =
  if (from > to) 1
  else f(from) * product(f)(from + 1, to)
```

两者代码非常相似，不同点有两处：0 或 1，+ 或 *，将这两处差异抽离为参数：

```Scala
def combine(f: Int ⇒ Int, op: (Int, Int) ⇒ Int, zero: Int)(from: Int, to: Int): Int =
  if (from > to) zero
  else op(f(from), combine(f, op, zero)(from + 1, to))
```

借助 `combine` 可以重新定义 `sum` 和 `product`：

```Scala
def sum(f: Int ⇒ Int)(from: Int, to: Int): Int =
  combine(f, _ + _, 0)(from, to)

def product(f: Int ⇒ Int)(from: Int, to: Int): Int =
  combine(f, _ * _, 1)(from, to)
```

## 总结

最后代码为：

```Scala
def combine(f: Int ⇒ Int, op: (Int, Int) ⇒ Int, zero: Int)(from: Int, to: Int): Int =
  if (from > to) zero
  else op(f(from), combine(f, op, zero)(from + 1, to))

def sum(f: Int ⇒ Int)(from: Int, to: Int): Int =
  combine(f, _ + _, 0)(from, to)

def product(f: Int ⇒ Int)(from: Int, to: Int): Int =
  combine(f, _ * _, 1)(from, to)

def fact(n: Int): Int = if (n == 0) 1 else n * fact(n - 1)

def sumInts: (Int, Int) ⇒ Int = sum(x ⇒ x)
def sumCubes: (Int, Int) ⇒ Int  = sum(x ⇒ x * x * x)
def sumFacts: (Int, Int) ⇒ Int  = sum(fact)
```

我们通过不断抽象，逐渐提升了函数的适用范围，函数越抽象，适用范围越大。

`combine` 没有任何代码适用了 `Int` 的特殊性质，因此实际上 `combine` 适用于任意 `Monoid`，加入 `Monoid`，并将区间改为 `List`：

```Scala
def combine[A: Monoid](f: A ⇒ A, op: (A, A) ⇒ A, zero: A)(xs: List[A]): A = {
  val m = implicitly[Monoid[A]]
  xs.foldLeft(m.zero)((b, a) ⇒ m.op(b, f(a)))
}
```

借助 `Monoid` 等 type class 概念，可以获得更加抽象、更加通用的 `sum` 函数，更多内容可参考另一篇文章 {% post_link 2018-06-03-scala-cats-sum-by-monoid Type Class | 基于 Monoid 和 Foldable 的 sum 函数 %}。

----

参考：

* [Functional Programming Principles in Scala week 2](https://www.coursera.org/learn/progfun1/lecture/xuM1M/lecture-2-1-higher-order-functions)