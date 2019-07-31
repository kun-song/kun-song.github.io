---
title: 传值调用 vs 传名调用
tags:
  - 求值策略
categories: Scala
abbrlink: 15180
date: 2018-05-17 22:00:17
---

要理解传值调用和传名调用到底是什么，以及他们有什么区别，首先要懂一点理论。

## 求值策略

编程语言的求值策略决定：

* 什么值可以传递给函数（what kind of value to pass to the function）；
* 什么时候对函数调用的 **参数求值**（when to evaluate the argument(s) of a function call）；

<!-- more -->

不同语言具有不同的求值策略，例如：

* C#、Java 采用 call by value/call by reference
* C++ 有多种求值策略
* 古老的 ALGOL 60 同时采用 call by value 和 call by name
* PL/I 采用 call by reference
* Haskell、R 采用 call by need

求值策略可以大致分为两类：

* 严格求值（strict evaluation），也称为 eager evaluation
* 非严格求值（non-strict evaluation），也称为 lazy evaluation

### 严格求值

严格求值中，函数应用（调用）之前，必须 **先对参数求值**。

目前大部分编程语言都采用严格求值，严格求值还可以分为很多子类：

* call by value
* call by reference
* call by sharing
* call by copy-restore
* partial evaluation

Scala 采用了传值调用（call by value），所以简单介绍一下：

传值调用中，首先计算参数表达式，然后将其计算结果绑定到函数的形式参数上（一般通过拷贝结果值），以便函数体使用。

传值调用仅规定先计算参数，并没有限制计算顺序：

* Java、Common Lisp、Eiffel 从左到右计算
* 有的语言从右到左计算
* Scheme、OCaml、C 没有规定顺序

### 非严格求值

非严格求值中，只有当函数体内 **实际使用** 特定参数时，才会对该参数求值。

非严格求值同样有很多子类：

* call by name
* call by need
* call by macro expansion

Scala 也采用了传名调用（call by name），也简单介绍一下：

传名调用中，不会先对参数求值，而是将函数体中的参数引用 **替换** 为实际参数，然后对函数体求值，若实际参数在函数体中出现，则对其求值，从而造成：

* 若函数体没有使用特定参数，则 **不会** 对其求值；
* 若函数体 **多次使用** 特定参数，则会多次 **重复求值**；

传名调用（call by name）在某些场景中比传值调用（call by value）更佳：

1. 函数体根本没有使用某些参数；
2. 参数为 **无法终止** 的计算；

但传名参数需要借助 [thunk](https://en.wikipedia.org/wiki/Thunk#Call_by_name) 实现，效率慢一点。

>传名调用最早由 ALGOL 60 采用。

## Scala 求值策略

大部分编程语言仅支持一种 [求值策略](https://en.wikipedia.org/wiki/Evaluation_strategy)，但 Scala 同时支持两种：

* 传值调用（call by value）
* 传名调用（call by name）

### 传值调用

Scala 参数默认即为传值调用，例如：

```Scala
def f(s: String): String = s + s
```

使用 `{ println("--"); "Hello" }` 表达式调用该函数：

```Scala
// --
// HelloHello
f({ println("--"); "Hello" })
```

先计算参数表达式，即 `{ println("--"); "Hello" }`，结果为 `Hello` 并打印 `--` 到标准输出，`f` 函数体内 `s` 即引用 `Hello`，所以最终输出一次 `--`。

### 传名调用

在参数类型前加 `⇒` 即可将其修改为传名调用：

```Scala
def f(s: ⇒ String): String = s + s
```

再次以 `{ println("--"); "Hello" }` 表达式调用 `f`，此时不会先计算参数，而是将 `f` 函数体内的 `s` 替换为 `{ println("--"); "Hello" }` 表达式，替换后 `f` 函数体为：

```Scala
{ println("--"); "Hello" } + { println("--"); "Hello" }
```

最后输出两次 `--`。

传名调用的缺点很明显：若参数被使用多次，会出现 **重复计算**，Scala 提供 `lazy` 关键字以 **显式** 缓存计算结果：

```Scala
def f(s: ⇒ String): String = {
  lazy val ss = s  // 此时不会对 s 求值

  // 第一个 ss 会触发求值，第二个 ss 将使用缓存值
  ss + ss
}
```

>`lazy` 将延迟对 `s` 表达式的求值，直到它被 **第一次引用**，之后会 **缓存第一次求值** 的结果，以后的引用不再重复计算。

因此 `f({ println("--"); "Hello" })` 执行时，`{ println("--"); "Hello" }` 只在 `ss + ss` 中第一个 `ss` 引用它时才被计算一次，第二个 `ss` 将使用第一次计算的缓存值，因此只输出一次 `--`。

>支持传名调用有什么意义呢？
>* 在不支持头等函数的语言中，传值调用与传名调用区别不大；
>* 在支持头等函数的语言中，计算量巨大的函数可以作为参数传递，此时适合用传名调用；

### 手动实现传名调用

若不使用 `⇒` 和 `lazy`，也能轻易实现传名调用的效果，秘诀是 **将参数值包裹在无参函数体中**：

```Scala
def f(s: () ⇒ String): String = {
  val ss = s()  // 类似 lazy val ss = s
  ss
}
```

使用匿名函数调用：

```Scala
f(() ⇒ { println("--"); "Hello" })
```

效果与之前完全相同！

实际上，`⇒` 和 `lazy` 仅仅是 [thunk](https://en.wikipedia.org/wiki/Thunk#Call_by_name) 的语法糖而已，我们完全可以手动实现传名参数，只是 Scala 认为手动版本有点繁琐，于是提供了一些语法糖方便用户而已。

>关于用 thunk 实现延迟计算的更多细节，可以参考 [Programming Languages Part B](https://www.coursera.org/learn/programming-languages-part-b/home/welcome) 第一周内容，讲解清晰易懂。

## 来自 Databricks 的建议

Databricks 建议 [避免使用传名调用](https://github.com/databricks/scala-style-guide/blob/master/README-ZH.md#call_by_name)，推荐使用显式函数参数 `() => T`。

因为在参数使用处，**无法区分** 该参数是传值调用，还是传名调用，因此无法确定该参数引用的表达式是否会执行，以及是否会执行多次，对于带有 **副作用** 的表达式而言，这非常危险。

---

参考：

* 维基百科 [Thunk](https://en.wikipedia.org/wiki/Thunk#Call_by_name)
* 维基百科 [Evaluation strategy](https://en.wikipedia.org/wiki/Evaluation_strategy#Non-strict_evaluation)
* [Databricks Scala 编程风格指南](https://github.com/databricks/scala-style-guide/blob/master/README-ZH.md)