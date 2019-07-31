---
title: Scala | 偏函数
tags:
  - 偏函数
categories: Scala
abbrlink: 48567
date: 2018-05-15 23:08:20
---

## 基本概念

偏函数，即 partial function，是一个数学上的概念，与 total function 对应，详细定义可以参考维基百科 [Partial Function](https://en.wikipedia.org/wiki/Partial_function) 条目。

简单地说，对于类型为 `A => B` 的函数 `f` 而言：

* 若 `f` 能处理所有类型 `A` 的值，则 `f` 为 total function；
* 若 `f` 仅能处理类型为 `A` 的值的 **子集**，则 `f` 为 partial function；

<!-- more -->

看一个具体例子，假设有两个类型为 `Double ⇒ Double` 的函数：

```Scala
val addOne: Double ⇒ Double = x ⇒ x + 1

val sqrt: Double ⇒ Double = x ⇒ Math.sqrt(x)
```

其中 `sqrt` 根本无法处理非正实数，因此 `sqrt` 是数学上的 partial function。

## 创建 `PartialFunction`

Scala 使用 `PartialFunction` 类型表示偏函数，`sqrt` 可以实现为 `PartialFunction` 实例：

```Scala
val sqrt: PartialFunction[Double, Double] =
  new PartialFunction[Double, Double] {
    override def isDefinedAt(x: Double) = x >= 0
    override def apply(v1: Double) = Math.sqrt(v1)
  }
```

Scala 将函数视为第一等公民，可 `PartialFunction` 使用也太麻烦了吧？为了方便用户，Scala 允许通过 `case` 关键字创建 `PartialFunction` 函数的字面值

```Scala
val sqrt: PartialFunction[Double, Double] = {
  case x if x >= 0 ⇒ Math.sqrt(x)
}
```

是不是清爽了很多，虽然不如普通函数字面值简洁，但也相去不远了。

>`case` 语句仅仅是语法糖而已，Scala 编译器会将其转换为前面的 `PartialFunction` 实例创建代码。

## 组合 `PartialFunction`

可以用 `orElse` 组合偏函数，若有如下偏函数：

```Scala
val one: PartialFunction[Int, String] = {
  case 1  ⇒ "one"
}

val two: PartialFunction[Int, String] = {
  case 2  ⇒ "two"
}

val three: PartialFunction[Int, String] = {
  case 3  ⇒ "three"
}
```

我们想要一个能同时翻译 1, 2, 3 的函数，可以这样组合：

```Scala
val all = one orElse two orElse three
```

`all` 函数可以处理 1 2 3：

```Scala
all(1)  // one
all(2)  // two
all(3)  // three
```

## 偏函数应用

在 Scala 代码中，偏函数几乎随处可见，本文仅列举几个简单例子。

### `List.collect`

对列表中每个元素加 1，可以用 `map` 轻易实现：

```Scala
List(1, 2, 3, 4) map {
  case n: Int ⇒ n + 1
}
```

但如果 `List` 中含有非数字类型，则 `map` 将抛出异常：

```Scala
// 抛出异常
List(1, 2, 3, "Hi") map {
  case n: Int ⇒ n + 1
}
```

此时可以用 `collect` 函数取代 `map`：

```Scala
List(1, 2, 3, "Hi") collect {
  case n: Int ⇒ n + 1
}
```

结果为 `List(2, 3, 4)`，跳过了字符串 `Hi`，原因是 `collect` 参数为 `PartialFunction`，其实现根据偏函数参数的定义域，自动跳过无定义的元素。

### Akka 中的例子

偏函数在 Akka 中使用非常普遍，例如前几天我提的这个 [PR](https://github.com/akka/akka/pull/25036/files) 中：

```Scala
def recoverWith(clazz: Class[_ <: Throwable], supplier: Supplier[Graph[SourceShape[Out], NotUsed]]): javadsl.Flow[In, Out, Mat] =
  recoverWith {
    case elem if clazz.isInstance(elem) ⇒ supplier.get()
  }
```

`recoverWith` 参数是偏函数：

```Scala
PartialFunction[Throwable, Graph[SourceShape[Out], NotUsed]]
```

通过偏函数，清晰表明 `recoverWith` 并非能处理所有 `Throwable` 值，而是仅处理参数 `PartialFunction` 能处理的 `Throwable` 子集。

有人可能会问，即使不用偏函数，我用 `Throwable ⇒ Option[Graph[SourceShape[Out], NotUsed]]` 一样能表达相同的语义啊？哈哈，原因很简单，偏函数也是函数，可组合性更好，更方便。

---

参考

* [Effective Scala](http://twitter.github.io/effectivescala/#Functional%20programming-Partial%20functions)
* [Twitter Scala School](https://twitter.github.io/scala_school/zh_cn/pattern-matching-and-functional-composition.html)