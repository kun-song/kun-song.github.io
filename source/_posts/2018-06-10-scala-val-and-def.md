---
title: Scala | 求值策略：val vs def
date: 2018-06-10 20:29:05
tags:
  - 求值策略
  - val
  - def
categories: Scala
---

在 {% post_link 2018-05-17-call-by-value-call-by-name 传值调用 vs 传名调用 %} 一文中我以函数参数为例介绍了 Scala 的两种求值策略：

* call by value
* call by name

实际上求值策略并不局限于参数参数。

<!-- more -->

Scala 有两种定义值（value definitions）的方式：

* `def`
* `val`

它们分别于两种求值策略一一对应：

* `def` 本质为 by name，因为每次对 `def` 定义的值 **求值时**，都会计算一次其右侧部分。
* `val` 本质为 by value，因为使用 `val` **定义值时**，其右侧部分马上进行计算，并将计算结果绑定到 `val` 左侧的名字上。

## `def`

```Scala
def x = 2
def y = square(x)
```

`x` 绑定到表达式 `2`，但不会马上对其求值；`y` 绑定到表达式 `square(2)`，同样不会立刻求值，只有需要对 `x` 和 `y` 求值时，才会真正开始计算它们代表的表达式：

```Scala
x + y
```

## `val`

```Scala
val x = 2
val y = square(x)
```

使用 `val` 定义时，立即对右侧部分求值，因此 `y` 绑定到 4，而不是 `square(2)`。

## 区别

当右侧表达式 **无法终止** 时，`val` 和 `def` 区别更加明显。

>下面操作都在 repl 中进行。

首先定义一个无法终止的函数：

```Scala
def loop: Boolean = loop
```

若用 `def` 则表达式立即结束：

```Scala
def x = loop
```

若用 `val` 则卡死在 `val` 定义：

```Scala
val x = loop
```

## 练习

实现 `and` 和 `or` 函数，使其对于任意表达式参数都能满足：

```Scala
and(x, y) == x && y
or(x, y)  == x || y
```

`&&` 和 `||` 都是短路求值，将第二个参数声明为 call by name 参数即可：

```Scala
def and(x: Boolean, y: ⇒ Boolean): Boolean = if (x) y else false
def or(x: Boolean, y: ⇒ Boolean): Boolean = if (x) true else y 
```

测试：

```Scala
def loop: Boolean = loop

and(false, loop)
and(true, loop)  // 死循环

or(true, loop)
or(false, loop)  // 死循环
```
