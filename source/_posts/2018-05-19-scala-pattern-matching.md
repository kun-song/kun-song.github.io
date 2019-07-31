---
title: Scala | 模式匹配
tags:
  - 模式匹配
categories: Scala
abbrlink: 47388
date: 2018-05-19 22:16:37
---

模式匹配是一种将值（value）与模式（pattern）进行匹配的机制，匹配成功还可以对值进行 **析构**，非常强大。

模式匹配在函数式编程语言中广泛存在，例如 Haskell、ML、OCaml、Erlang 和 Elixir 等，Scala 也不例外。

<!-- more -->

## 基本语法

模式匹配由 3 部分组成：

1. 被配置的值（或表达式）
2. `match` 关键字
3. `case` 子句（也称 alternatives）

例如：

```Scala
def translate(x: Int): String = x match {
  case 0 ⇒ "zero"
  case 1 ⇒ "one"
  case 2 ⇒ "two"
  case _ ⇒ "many"
}
```

* 模式匹配是表达式，因此也有值

case 子句格式为：

```Scala
模式 ⇒ 表达式
```

Scala 提供了多种匹配模式，下面一一介绍。

## 字面值（常量）模式

匹配 `Int` 字面值：

```Scala
def translate(n: Int): String = n match {
  case 1 ⇒ "one"
  case 2 ⇒ "two"
  case 3 ⇒ "three"
}
```

## 通配符模式

`translate` 只能处理 1-3 的整数，`translate(4)` 会抛出 `MatchError` 错误，可用通配符解决：

```Scala
def translate(n: Int): String = n match {
  case 1 ⇒ "one"
  case 2 ⇒ "two"
  case 3 ⇒ "three"
  case _ ⇒ "many"
}
```

* 通配符 `_` 将匹配除 1 2 3 以外的所有整数

## 类型模式

根据值的类型进行匹配：

```Scala
def recover(cause: Throwable): String = cause match {
  case _: IllegalArgumentException ⇒ "recover from IllegalArgumentException"
  case _: NullPointerException     ⇒ "recover from NullPointerException"
  case _: ArithmeticException      ⇒ "recover from ArithmeticException"
  case e                           ⇒ s"sorry, I can't recover from $e"
}
```

使用：

```Scala
// recover from IllegalArgumentException
recover(new IllegalArgumentException)
// sorry, I can't recover from java.lang.RuntimeException
recover(new RuntimeException)
```

## 元组模式

元组：

```Scala
def check(info: (String, Int)): Boolean = info match {
  case (name, _) ⇒ !name.isEmpty
  case (_, age)  ⇒ age >= 0
}
```

## 序列模式

列表：

```Scala
def sum(xs: List[Int]): Int = xs match {
  case Nil      ⇒ 0
  case hd :: tl ⇒ hd + sum(tl)
}
```

## 守卫

可通过守卫进一步 **约束** 模式匹配：

```Scala
def check(hostAndPort: (String, Int)): String = hostAndPort match {
  case (_, port) if port == 0 ⇒ "Use a random port"
  case (host, port)           ⇒ s"Use $host:$port"
}

check(("localhost", 0))
check(("127.0.0.1", 27017))
```

## 构造器模式

构造器模式专用于 case 类：

```Scala
abstract class Person
case class Student(name: String, age: Int) extends Person
case class Worker(name: String, age: Int) extends Person

def f(person: Person): String = person match {
  case Student(name, age) ⇒ s"$name is a $age years old student!"
  case Worker(name, age)  ⇒ s"$name is a $age years old worker!"
}
```

* 首先匹配 case 类的类型，然后匹配参数

## 变量绑定

// TODO

## 提取器

// TODO

## 模式匹配中的变量、常量

// TODO

## 与 switch 语句区别

对于 Java 背景的读者而言，模式匹配可以视为加强版的 switch 语句，但有如下区别：

* 模式匹配中的 `match` 语句是表达式，有计算结果
* 模式匹配中的 case 子句无需手动 break
* 配合封闭类，Scala 编译器可以保证模式匹配 **无遗漏分支**

>实际上，模式匹配仅仅是恰好能完成 switch 的功能而已，二者概念上并无太大关系。


