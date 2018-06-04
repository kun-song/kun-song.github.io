---
title: 为什么需要尾递归？
tags:
  - Scala
  - fp
  - 尾递归
categories: 技术
abbrlink: 32377
date: 2018-05-16 22:58:23
---

## 尾调用

若函数 **最后一步** 操作为函数调用，则称之为尾调用（tail call）。

尾调用的关键是最后一步必须是“纯粹”的函数调用，不能有任何其他操作，例如：

```Scala
def sum(xs: List[Int]): Int =
  if (xs.isEmpty) 0
  else xs.head + sum(xs.tail)
```

* `sum` 函数最后一步 `xs.head + sum(xs.tail)` 中除调用 `sum` 函数外，还做了加法，因此不是尾调用

<!-- more -->

稍作修改下就是尾调用了：

```Scala
def sum(xs: List[Int], ans: Int = 0): Int =
  if (xs.isEmpty) ans
  else sum(xs.tail, ans + xs.head)
```

## 尾调用消除

一般函数调用会在 **调用栈**（call stack）上压入当前函数的 **栈帧**（stack frame），因此递归调用将 **层层压栈**，若调用层次太多可能造成栈溢出。

尾调用 **最后一步** 是调用其他函数，被调用函数的结果即为当前函数的结果，因此可以不必保留当前函数的栈信息，仅保留尾调用函数的栈即可，因此可保持 **常数级** 的栈空间消耗，从而避免栈溢出错误，这被称为尾调用消除（tail call elimination）。

## 尾递归

若尾调用的函数为自身，则称之为尾递归，尾递归是一种特殊的递归函数。

函数式编程语言推崇使用递归解决问题（部分函数式编程语言甚至不支持迭代，只能使用递归），因为相对迭代（循环）而言，递归更适合人脑思考，更加自然，但递归非常消耗内存，很容易出现栈溢出问题。

但若是尾递归则可借助“尾调用消除”来优化栈空间使用，从而达到与迭代接近的内存消耗，进而一举获得递归、迭代两者的优点。

>已经证明，所有递归函数都可以被改写为迭代方式，且尾递归可以按照 **固定步骤** 改写为迭代，所以很多编程语言会 **自动** 执行尾递归优化。

## Scala 中的尾递归

Scala 是函数式编程语言，自然也推崇递归函数，且 Scala 语言规范规定，必须实现对尾递归的优化（受限于 JVM，未实现尾调用优化），Scala 编译器会自动识别尾递归函数，并进行优化。

有时编译器不一定能准确识别所有尾递归函数，因此 Scala 提供 `@tailrec` 注解，用来标识需要被优化的尾递归函数，例如：

```Scala
import scala.annotation.tailrec

def sum(xs: List[Int]): Int = {
  @tailrec
  def aux(l: List[Int], acc: Int): Int =
    if (l.isEmpty) acc
    else aux(l.tail, acc + l.head)

  aux(xs, 0)
}
```

* 若 `@tailrec` 修饰的函数不是尾递归，则编译报错

>这个例子也揭示了将普通递归函数改写为尾递归函数的方式，即将结果保存在 **参数** 中，这是一种非常常见的优化手段。

注意，Scala 并未实现对 **普通递归** 函数的优化，因此普通递归依然存在栈溢出的风险，例如：

```Scala
def fact(n: Int): Int =
  if (n == 0) 1
  else n * fact(n - 1)
```

`fact` 是一个非常符合直觉的的递归解法，不幸的是 `n` 稍微大一点就会出现栈溢出，例如 `fact(100000)`。

幸运的是我们可以将其改写为尾递归形式，从而消除栈溢出风险，改写方式与 `sum` 相同，都是通过将结果保存在参数中实现：

```Scala
def fact(n: BigInt): BigInt = {
  @tailrec
  def aux(n: BigInt, acc: BigInt): BigInt =
    if (n == 0) acc
    else aux(n - 1, acc * n)

  aux(n, 1)
}
```

* 为避免截断，将 `Int` 改为 `BigInt`

优化之后的 `fact` 将使用常数级的栈空间，因此 `fact(100000)` 可以正确计算（结果非常大，`Int` 无法容纳），不会出现栈溢出。

关于更多普通递归改写为尾递归的例子，可以参考 [Programming Languages Part A](https://www.coursera.org/learn/programming-languages/home/welcome)，如果你觉得还不习惯这种改写方式，强烈建议跟下这门公开课。

## 来自 Databricks 的建议

关于是否使用递归，历来争议很大，Databricks 根据自己开发维护 Spark 的经验给出的 [建议](https://github.com/databricks/scala-style-guide/blob/master/README-ZH.md#recursion) 是：

>**避免使用递归**，除非问题可以非常自然地用递归描述（如树、图的遍历）。

并且建议为每个尾递归函数添加 `@tailrec` 注解，因为闭包、函数变换的使用，许多 **看似** 尾递归的函数实际上并非是尾递归，使用 `@tailrec` **强制** 编译器检查函数是否是尾递归是个好习惯。

为什么不推荐使用递归呢，Databricks 给出的理由是：大多数代码使用迭代（循环）和状态机更容易推理，而递归反而难以理解。

证明其观点的例子是：

```Scala
// 尾递归
def max(data: Array[Int]): Int = {
  @tailrec
  def max0(data: Array[Int], pos: Int, max: Int): Int =
    if (pos == data.length) max
    else max0(data, pos + 1, if (data(pos) > max) data(pos) else max)
  
  max0(data, 0, Int.MinValue)
}

// 迭代
def max(data: Array[Int]): Int = {
  var max = Int.MinValue
  for (v <- data) {
    if (v > max) max = v
  }
  max
}
```

当然 Databricks 的观点并非金科玉律，代码要易于维护，至于选择迭代还是递归，完全取决于使用场景。

---

参考：

* 阮一峰博客 [尾调用](http://www.ruanyifeng.com/blog/2015/04/tail-call.html)
* 维基百科 [Tail call](https://en.wikipedia.org/wiki/Tail_call)
* [Databricks Scala 编程风格指南](https://github.com/databricks/scala-style-guide/blob/master/README-ZH.md)