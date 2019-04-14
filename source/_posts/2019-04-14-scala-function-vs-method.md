---
title: Scala | 函数 vs 方法
date: 2019-04-14 13:50:03
tags:
  - Scala
  - eta-expansion
categories: 技术
---

在 Scala 中，函数（function）和方法（method）是两个非常接近的概念，大多数场景中，可以不区分两者，如果非要区分，可查阅文末的参考文档。

根据 [官方教程](https://docs.scala-lang.org/tour/basics.html#functions)：

* 函数分两类：
  + 实现 `FunctionN` 的对象
  + 匿名函数
* 方法：在类、对象内部用 `def` 声明

<!-- more -->

匿名函数的类型实际也是 `FunctionN`，因此可以认为，在 Scala 中，函数就是实现 `FunctionN` 特质的对象，函数调用 `f(x)` 实际是以下 `apply` 方法调用的 **语法糖**：

```Scala
f.apply(x)
```

方法与函数只有一层关系：

>大部分方法可以通过 eta expansion 转换为实现 `FunctionN` 特质对象，即函数。

在需要函数的地方，方法实际被转换为实现了 `FunctionN` 特质的对象，从而实现与函数基本相同的行为，例如：

```Scala
def addOne(x: Int): Int = x + 1

def process(f: Int => Int, x: Int): Int = f(x)
process(addOne, 1) // 2
```

`process` 需要一个 **函数** 作为参数，而 `addOne` 是一个 **方法**，因此 `addOne` 实际被转换为：

```Scala
new Function1[Int, Int] {
  def apply(x: Int): Int = addOne(x)
}
```

这个函数对象自然可以用在 `process` 中。

>还可以显式进行 method -> function 的转换，即 `methodName _`。

当然并非所有 `def` 定义的方法都会被转换为函数：

```Scala
class Person {
  def hello1: String = "hello"
  val hello2 = () => "hello"
}

val p = new Person
```

调用区别：

```Scala
p.hello1 // res0: String = hello
p.hello2 // res1: () => String = Person<function>

p.hello1() // 报错 
p.hello2() / res2: String = hello
```

---

参考：

* [Difference between method and function in Scala](https://stackoverflow.com/questions/2529184/difference-between-method-and-function-in-scala)
* [Scala Functions vs Methods](http://jim-mcbeath.blogspot.com/2009/05/scala-functions-vs-methods.html)
* [What is the eta expansion in Scala?](https://stackoverflow.com/questions/39445018/what-is-the-eta-expansion-in-scala)

