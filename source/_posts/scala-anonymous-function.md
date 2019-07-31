---
title: 匿名函数
tags:
categories: Scala
abbrlink: 58244
date: 2018-05-16 08:42:18
---

匿名函数（anonymous function），也称函数字面值（function literal）、lambda 抽象、lambda 表达式，顾名思义，就是没有名字的函数值。

本文主要讲解 Scala 中的匿名函数，最后简单介绍 Java 对匿名函数的支持。

## 字面值

在继续讲解匿名函数之前，先回顾一下字面值（literal）的概念。虽然我们工作中几乎每时每刻都在跟字面值打交道，但大家对这个习以为常的概念却并一定清楚。

<!-- more -->

维基百科对字面值（literal）解释如下：

>In computer science, a literal is a notation for representing a **fixed value** in source code. Almost all programming languages have notations for atomic values such as integers, floating-point numbers, and strings, and usually for booleans and characters; some also have notations for elements of enumerated types and compound values such as arrays, records, and objects. An anonymous function is a literal for the function type.

在编程语言中，某些类型被认为 **很重要**，为方便使用，语言会提供表示该类型的固定值的语法糖，例如 Scala 为 `Int`，`Float`，`String` 等类型提供了字面值：

* `11` 是整数字面值，类型为 `Int`；
* `'a'` 是字符字面值，类型为 `Char`；

Scala 认为函数（`FunctionN`）也是很重要的类型，因此提供了函数字面值：

* `(x: Int) ⇒ x + 1` 是函数字面值，类型为 `Function1[Int, Int]`（`Int ⇒ Int`）；

字面值一般用作匿名值（anonymous values），即不需要先将其绑定到变量就能使用，在不需要 **复用** 字面值的使用场景中，该特性使代码非常简洁：

```Scala
List(1, 2, 3).filter(x ⇒ x > 2)
```

若字面值不能用作匿名值，则上述代码只能写作：

```Scala
val one: Int = 1
val two: Int = 2
val three: Int = 3
val greaterThan2: (Int ⇒ Int) = (x: Int) ⇒ x > two
List(one, two, three).filter(greaterThan2)
```

这个例子似乎有点荒诞，毕竟谁会这样写代码啊！事实上，如果没有字面值，你就得这么写。因为字面值太常见了，大家都已经习以为常了，所以这里帮大家重温一下基本概念。

## Scala 匿名函数

Scala 提供了非常轻量级的匿名函数创建语法，例如：

```Scala
// Int ⇒ Int
(x: Int) ⇒ x + 1
```

>匿名函数的创建语法，仅仅是创建 `FunctionN` 类型实例的语法糖，上面的匿名函数等价于：
```Scala
new Function1[Int, Int] {  
 def apply(x:Int): Int = x + 1;  
}
```

可以将匿名函数绑定到名字：

```Scala
val addOne = (x: Int) ⇒ x + 1
```

匿名函数 **最常见** 的用法是作为高阶函数的参数（高阶函数 + 匿名函数，焦不离孟）：

```Scala
val xs = List.range(0, 10)
xs filter ((n: Int) ⇒ n % 2 == 0)  // List(0, 2, 4, 6, 8)
```

* `(n: Int) ⇒ n % 2 == 0` 函数用完就扔，因此不需要名字，匿名函数正合适。

因为 `(n: Int) ⇒ n % 2 == 0` 只有一个入参，因此可以简化为：

```Scala
xs filter (_ % 2 == 0)  // List(0, 2, 4, 6, 8)
```

除了创建普通函数（`FunctionN`）的字面值以外，还可以创建 `PartialFunction` 类型的字面值：

```Scala
val sqrt: PartialFunction[Double, Double] = {
  case x if x >= 0 ⇒ Math.sqrt(x)
}
```

>注：这种说法不确实是否准确，Twitter Scala School 中认为 `case` 语句是 `PartialFunction` 的子类。

## Java 匿名函数（lambda 表达式）

Java 8 通过 lambda 表达式提供对匿名函数的支持，用法与 Scala 匿名函数非常类似：

```Java
(long id, String name) -> "id: " + id + ", name:" + name
```

* lambda 表达式会被转化为 **函数接口**

还可以通过 **方法引用** 创建 lambda 表达式：

```Java
IntBinaryOperator sum = Integer::sum;
```

Java 8 lambda 表达式有两个缺点：

1. lambda 表达式可以抛出 checked exception，且若抛出 checked exception 则不能用于 `filter` `map` 等集合 API
2. lambda 表达式只能使用 effectively final 非局部变量（自由变量）

以上限制看似微不足道，实际上大大削弱了 lambda 表达式的灵活性，而 Scala 匿名函数则无类似缺点。

## 总结

* 匿名函数（函数字面值）之于 `FunctionN`，如同 `5` 之于 `Int`
* 匿名函数一般用作：
  + 高阶函数的参数
  + 单行的辅助函数
* 匿名函数是创建 `FunctionN` 实例的语法糖

---

参考：

* 维基百科 [Anonymout function](https://en.wikipedia.org/wiki/Anonymous_function)