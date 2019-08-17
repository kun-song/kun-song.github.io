---
title: Algebraic Data Type 及其在 Haskell 和 Scala 中的表现
date: 2018-07-12 00:40:11
  - Algebraic Data Type 
categories: Scala
---

函数式编程接触久了以后，我们会发现很多 FP 语言（这里指静态 FP 语言）具有不少类似的语言特性，这非常自然，因为语言特性就那么多，好用、实用的特性更少，这一方面造成了语言之间的同质化，另一方面也减轻了我们语言切换的成本，算是有利也有弊吧。

常见的静态函数式语言有 Haskell、Standard ML、OCaml、Scala 等，它们之间非常类似，共有的特性有：

* first-class function（匿名函数、高阶函数、闭包、柯里化、部分应用函数...）
* algebraic data type
* 模式匹配
* 类型推断

各个语言也有自己相对独特的特性，比如 Haskell 的 typeclass，Standard ML 和 OCaml 的模块，Scala 的私货就更多了，为了兼容 OO 和 FP 做了很多设计上的妥协，也引入了很多概念。

<!-- more -->

>函数式编程语言也有动态、静态之分，上面列举的都是静态语言，动态语言则以 Lisp 为代表，包括：Scheme、Racket、Clojure、Erlang、Elixir 等，由于动态类型的限制，它们没有 algebraic data type、类型推断等类型系统相关的特性。

说了这么多，只是想告诉大家，ADT（algebraic data type）真真是 FP 语言中非常常见的概念，另外 ADT 与模式匹配可谓焦不离孟，孟不离焦，搭配起来威力强大。

## 什么是 ADT ？

说了这么多，到底什么是 ADT 呢？

不要着急，我们知道编程语言会内置一些[基本类型](https://en.wikipedia.org/wiki/Primitive_data_type)，例如 `int`、`String` 等，基本类型只能表示非常简单的值，无法对复杂的现实世界建模，因此编程语言还会提供将基本类型 **组合** 成复杂类型机制，这些组合出来的类型被称为[组合类型](https://en.wikipedia.org/wiki/Composite_data_type)，维基百科对组合类型的定义为：

In computer science, a composite data type or compound data type is any data type which can be constructed in a program using the programming language's primitive data types and other composite types.

>注：编程语言中的两种类型：
>* 基本类型
>* 组合类型

不同编程语言提供了不同的构建组合类型的机制，例如面向对象中的 class、struct 等，ADT 正是函数式编程语言构建组合类型的方式。

为什么叫 algebraic data type 呢，难不成还跟代数有关系？？？

Yes，还真和代数有关，根据 Haskell wiki：

"Algebraic" refers to the property that an Algebraic Data Type is created by "algebraic" operations. The "algebra" here is "sums" and "products":

* "sum" is alternation (A | B, meaning A or B but not both)
* "product" is combination (A B, meaning A and B **together)**

ADT 是通过 **代数运算** 构造的，常见的两种构造方式为：

* sum，即 a or b
* product，即 a and b

例如 `Tuple` 是由 product（a and b） 操作产生的，而 `Option` `Either` `Try`  是由 sum（a or b） 操作产生的。

## product 类型

product 类型是最常见的组合类型，class 在某种程度上也是 product 类型。

Scala 可以用 `case` 类构建 product 类型：

```Scala
case class Student(name: String, age: Int)
```

也可以用 `trait`：

```Scala
trait Student {
  def name: String
  def age: Int
}
```

而 Haskell 则用 record 语法构建：

```Haskell
data Student = Student { name:: String, age:: Int }
```

product 类型非常常见，常见到我们几乎已经忽略掉它了，俗称灯下黑。。。

## sum 类型

ADT 的强大之处更多体现在 sum 类型上，面向对象没有与之对应的特性（C/C++ 的 struct、Java 的 Enum 有点形似，但功能差太远）

Scala 通过 `sealed trait` + case 类构建 sum 类型：

```Scala
sealed trait Human

final case class Worker(name: String, company: String) extends Human
final case class Student(name: String, school: String) extends Human
```

而 Haskell 则更加简洁：

```Haskell
data Human = Worker String String | Student String String
```

两者语法有差异，但实际上非常神似：

* Haskell 中的 `Worker` 和 `Student` 都是 value constructor，则 Scala 中的 `Worker` 和 `Student` 也是 constuctor；
* Haskell 中定义的类型是 `Human`，而 Scala 虽然定义了 3 个不同类，但 `Worker` 和 `Student` 是 `Human` 的子类，因此可以认为仅定义了 `Human` 一个类型；
* Scala 通过 `sealed` 帮助编译器实现穷尽分析，而 Haskell 无需做任何额外工作，默认即获得穷尽分析的能力；

## 模式匹配

说到这里，大家肯定会想，ADT 也不过如此嘛，哪有什么神奇的地方？

好吧，ADT 本身的确没什么特别，ADT 的威力需要借助模式匹配才能释放出来，ADT + 模式匹配，这可能是我见过的最具有表现力、最实用、最强的语言特性了，非常赞！

模式匹配其实就做了两件事：

1. 匹配类型
2. 解构 ADT

例如：

```Haskell
sum :: [Int] -> Int
sum []       = 0
sum (x : xs) = x + sum xs
```

借助 ADT 和模式匹配，可以非常容易做解析：

```Haskell
desugar :: Statement -> DietStatement
desugar (Incr name)      = (DAssign name (Op (Var name) Plus (Val 1)))
desugar (For s1 e s2 s3) = (DSequence (desugar s1) (DWhile e (DSequence (desugar s3) (desugar s2))))
desugar (Assign name e)  = (DAssign name e)
desugar (If e s1 s2)     = (DIf e (desugar s1) (desugar s2))
desugar (While e s)      = (DWhile e (desugar s))
desugar (Sequence s1 s2) = (DSequence (desugar s1) (desugar s2))
desugar Skip             = DSkip
```

更多例子，可以参考 {% post_link 2018-06-24-haskell-a-tiny-interpreter A tiny interpreter in Haskell %}

---

参考：

* 网上一篇关于 Scala 中的 algebraic data type 的讲义：[slides](http://tpolecat.github.io/presentations/algebraic_types.html#17)
* Haskell wiki [Algebraic data type](https://wiki.haskell.org/Algebraic_data_type)
* 本人笔记 [数据模型](https://github.com/satansk/learning-fpinscala/wiki/%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B)
