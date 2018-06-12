---
title: 偏函数 vs 部分应用函数 vs 柯里化
tags:
  - Scala
  - fp
  - 偏函数
  - 部分应用函数
  - 柯里化
categories: 技术
abbrlink: 17257
date: 2018-05-16 00:19:56
---

三者的英文名字分别为：

* 偏函数：partial function
* 部分应用函数：partially applied function
* 柯里化：currying

其中偏函数、部分应用函数容易被混淆，部分应用函数、柯里化也容易被混淆，本文先介绍他们各自的概念，然后解释它们之间的区别。

<!-- more -->

## 基本概念

### 偏函数

`PartialFunction` 是数学上的概念，即定义在输入参数 **子集** 上的函数，例如：

```Scala
val squareRoot: PartialFunction[Double, Double] = {
  case n if n >= 0 ⇒ Math.sqrt(n)
}
```

`squareRoot` 参数类型为 `Double`，但只有 `Double` 的非负数子集才能调用 `squareRoot`。

### 部分应用函数

首先明确下术语，在函数式编程中，函数调用（a call to function that has parameters）的另一种叫法是：

>**applying** the function to the parameters.

因此：

* When a function is called with **all** the required parameters, it has **fully applied** the function to all of the parameters.
* But when only a **subset** of the parameters to the function is passed, the **result of the expression** is a partially applied function.

在 Scala 中，当函数调用的参数不足时，编译器不会报错，而是先 **应用** 已提供的参数，并返回一个 **新函数**，该函数接受原函数剩余未提供的参数作为自己的参数，称该函数为部分应用函数。

例如：

```Scala
val divide: (Double, Double) ⇒ Double = (x, y) ⇒ x / y
```

若想要计算给定数值的一半，则可以部分应用第二个参数：

```Scala
val halfOf = divide(_: Double, 2)

halfOf(10)  // 5.0
```

* 未提供的参数用 `_` 占位符表示；
* 注意，需要指明 `_` 的类型，要么在函数类型里指明，要么在参数类型里；

`halfOf` 为部分应用函数，其函数体类似：

```Scala
val halfOf: Double ⇒ Double = x ⇒ x / 2
```

### 柯里化

柯里化在部分应用函数基础上更进一步：

>Currying is the process of decomposing a function that takes **multiple** arguments into a sequence of functions, each with a **single** argument.

柯里化将多参函数分解为一系列的单参函数，例如：

```Scala
val divide: (Double, Double) ⇒ Double = (x, y) ⇒ x / y

// Double ⇒ (Double ⇒ Double)
val curriedDivide = divide.curried
```

`divide` 有两个参数，柯里化后得到 `curriedDivide`，该函数类型为 `Double ⇒ Double ⇒ Double`，其内部实现将 `divide` 分解为 **两个函数**：

1. `curriedDivide(10)` 调用第 1 个调用，并返回第 2 个函数；
2. `curriedDivide(10)(2)`，先后调用两个函数；

柯里化函数也是函数，因此可以对其部分应用：

```Scala
val halfOf = curriedDivide(_: Double)(2)

halfOf(10)
```

## 区别

了解各自的概念后，它们之间的区别已经一目了然了。

### 偏函数 vs 部分应用函数

虽然偏函数（partial function）、部分应用函数（partially applied function）的英文名字非常接近，但它们却是风马牛不相及的两个概念：

* 偏函数是仅能处理入参类型 **子集** 的函数，是数学概念；
* 部分应用函数是函数调用的结果，只不过该函数调用只提供了 **部分参数**；

### 部分应用函数 vs 柯里化

与前面一组不同，该两者并非由于名字接近而容易被混淆，部分应用函数和柯里化都可以用于 **减少函数参数**，但两者依然存在明显区别：

* 部分应用函数：应用已提供的参数，返回 **一个** 新函数，该函数接受剩余参数为参数；
* 柯里化：分解函数，将多参函数分解为 **一系列** 的单参函数；

部分应用函数仅产生一个函数，柯里化实际上可以产生多个函数。

若有函数：

```Scala
val add: (Double, Double, Double) => Double = (x, y, z) => z + y + z
```

可以部分应用 0 个参数：

```Scala
// (Double, Double, Double) => Double
val partialAdd = add(_: Double, _: Double, _: Double)
```

也可以将其柯里化：

```Scala
// Double => (Double => (Double => Double))
val curriedAdd = add.curried
```

部分应用函数的类型为 `(Double, Double, Double) => Double`，而柯里化后的类型为 `Double => (Double => (Double => Double))`，两者存在明显区别，一个是单个普通函数，一个是一系列单参函数。

通过多次部分应用，虽然繁琐一点，但可以达到与柯里化类似的效果：

```Scala
// (Double, Double) => Double
val partialAdd20 = partialAdd(_:Double, _: Double, 20)

// Double => (Double => Double)
val curriedAdd20 = curriedAdd(20)
```
