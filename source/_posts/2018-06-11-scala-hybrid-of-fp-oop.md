---
title: Scala 是如何融合 OO 和 FP 两种编程范式的 ？
date: 2018-06-11 21:17:29
tags:
categories: Scala
---

Scala 在其官网宣称：

>Scala combines object-oriented and functional programming in one concise, high-level language.

融合 OO 和 FP 并非稀奇事，OCaml、F# 都这么做过，毕竟 OO 和 FP 非常正交，一点都不冲突（见 [Are FP and OO orthogonal?](https://stackoverflow.com/questions/3949618/are-fp-and-oo-orthogonal)）。

融合两者并非易事，做不好容易给人一种割裂感，好在 Scala 在这点做的非常自然，基本做到了取长补短。

<!-- more -->

## 函数式 Scala

首先，作为 FP 语言，Scala 对函数支持非常完备：

* 便捷的匿名函数创建
* first class function、高阶函数
* 函数组合
* 闭包、柯里化、部分应用函数、偏函数
* laziness
* type class
* 模式匹配
* 类型推断
* ADTs
* 强大的类型系统（高阶类型、幻影类型、既存类型、类型上下界、上下文边界等等）

在 FP 方面，Scala 不输各种老牌 FP 语言，例如 ML、OCaml、Scheme 等，虽然语法啰嗦一点。

## 面向对象 Scala

其次，Scala 是一门 pure OO 语言，因为在 Scala 中一切皆对象，包括原始类型（如 `Int`、`Double` 等）。

那么问题来了，既然一切皆对象，那么函数又是什么？

出现该问题的根本原因是 Scala 对 OO 和 FP 的融合几乎天衣无缝、Scala 对 FP 支持如此自然，以致于大家产生这种疑问，毕竟函数看起来一点都不像对象！

## 函数式 vs 面向对象

然而事实是，函数也逃不出 OO 的手掌心：

1. 函数类型：为类
1. 函数值：为对象

### 函数类型：类

函数类型 `A => B` 不过是特质 `scala.Function1[A, B]` 的简写而已，`Function1[A, B]` 定义如下：

```Scala
trait Function1[-A, +B] {
  def apply(v1: A): B
}
```

>Scala 支持 `Function1` 到 `Function22`。

### 函数值：对象

而函数值仅仅是 `FunctionX` 类型的对象而已，更准确地说，函数值是具备 `apply` 方法的对象，只不过 `FunctionX` 恰好定义了 `apply` 而已。

例如函数值：

```Scala
(x: Int) => x * x
```

被编译器展开为：

```Scala
{
  class AnonFun extends Function1[Int, Int] {
    def apply(x: Int): Int = x * x
  }
  new AnonFunc
}
```

或者更简洁的：

```Scala
new Function1[Int, Int] {
  def apply(x: Int): Int = x * x
}
```

### 函数调用

好吧，函数值的确是对象，那函数调用呢，你怎么解释？？？

不要急，听我慢慢道来。

函数调用 `f(a, b)` 被编译器展开为 `f.apply(a, b)`（毕竟函数精华都在 `apply` 上）。

看一个完整的例子：

```Scala
val f = (x: Int) => x * x
f(11)
```

被展开为：

```Scala
val f = new Function1[Int, Int] {
  def apply(x: Int): Int = x * x
}
f.apply(11)
```

## eta 转换

细心的读者可能会问：你已经说服我函数（function）是对象了，那方法（method）呢，`apply` 方法是对象吗？

假设 `apply` 方法也是对象，因此也是 `Function1[Int, Int]` 的实例，代表它实例自己也有 `apply` 方法，进而又是 `Function1[Int, Int]` 的实例 ... 死循环了！

因此 `apply` 方法不可能是对象。

实际上，方法（使用 `def` 定义）本身 **并非函数值**：

```Scala
def f(n: Int): Int
```

但是，若 `f` 出现在需要函数类型的上下文中，`f` 被自动转换为函数值：

```Scala
(x: Int) => f(x)
```

或：

```Scala
new Function1[Int, Int] {
  def apply(x: Int): Int = f(x)
}
```

>将 method 转换为 function 的过程称为 eta 展开。

## 总结

* function value 是 object，而 function type 是 class
* 函数不过是具备 `apply` 方法的对象而已
* 方法不是函数，但可以转换为函数（eta 展开）

---

参考：

* [Are FP and OO orthogonal?](https://stackoverflow.com/questions/3949618/are-fp-and-oo-orthogonal)
* [Functional Programming Principles in Scala week 4](https://www.coursera.org/learn/progfun1/home/week/4)