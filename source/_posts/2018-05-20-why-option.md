---
title: 我们为什么需要 Option ？
tags:
  - Scala
  - fp
categories: 技术
abbrlink: 52452
date: 2018-05-20 21:43:42
---

Java 代码中，若方法无法返回有意义的值，有时贪图方便，会直接返回 `null`，并期待调用者自行对返回值判空。

这样做存在两个弊端：

1. 调用者无从得知方法是否会返回 `null`
2. Java 不强制 `null` 检查
3. 空指针检查是样板代码，会增加代码噪声

<!-- more -->

因此线上运行时，经常出现空指针异常，而遇到之后也不过头疼医头脚痛医脚，在出现问题的地方加个检查完事。

为避免空指针异常，Effective Java 建议若方法返回类型为 **集合**，则通过返回空集合以避免 `NullPointerException`。

若返回类型不是集合呢？又该用什么方式避免空指针呢？

Scala 给出的答案即为 `Option`（后面会看到 Java 8 引入了类似的 `Optional`）！

`Option` 是抽象代数类型，有两个子类，分别表示值存在和不存在：

* `Some`
* `None`

`Option` 相对手动空指针检查，有如下优势：

* 类型安全，**强制** 用户处理值不存在的场景，消除 `NullPointerException`
* 类型即文档，`Option` 明确表明结果可能不存在
* 可组合性（monadic）非常强

>要理解 `Option` 的可组合性并不容易，需要对 `Monad` 有所了解，可参考 [Scala with Cats](https://underscore.io/books/scala-with-cats/)。

## 创建

`Option` 创建非常简单，直接用伴生对象的 `apply` 工厂方法即可：

```Scala
// None
val a = Option(null)

// Some(100)
val b = Option(100)
```

若确认表达式不是 `null` 也可以直接用 `Some`：

```Scala
val c = Some(99)
```

* `Some(null)` 将产生一个包含 `null` 值的 `Some` 实例！

有了 `Option` 再也不用担心空指针异常了：

```Scala
def divide(x: Double, y: Double): Option[Double] =
  if (y == 0) None
  else Some(x / y)
```

再比如：

```Scala
def parseDouble(s: String): Option[Double] =
  try {
    Some(s.toDouble)
  } catch {
    case _: Throwable ⇒ None
  }
```

* 实际使用中，也可以用 `Try` 而非 `Option`

Scala 很多集合操作的返回类型即为 `Option`，例如 `Map.get`：

```Scala
val db = Map("Mike" → 20, "Bob" → 29, "Jane" → 19)

def findAgeByName(name: String): Option[Int] = db.get(name)
```

## `getOrElse`

可以用 `getOrElse` 方便地提取 `Option` 的值，并指定不存在时的默认值：

```Scala
val age = findAgeByName("Mike").getOrElse(-1)
```

## 模式匹配

因为 `Some` 和 `None` 都是 case 类，因此可以用模式匹配获取 `Option` 中的值：

```Scala
val age = findAgeByName("Mike") match {
  case Some(n) ⇒ n
  case None    ⇒ -1
}
```

模式匹配是大杀器，但用于提取 `Option` 值有点杀鸡用牛刀，`getOrElse` 要更加轻便。

## 组合操作

`map`：

```Scala
val x = divide(100, 2).map(_ + 1)  // Some(51)
val y = divide(100, 0).map(_ + 1)  // None
```

`flatMap`：

```Scala
val x = divide(100, 2).flatMap(v ⇒ divide(v, 2))  // Some(25)
val x = divide(100, 0).flatMap(v ⇒ divide(v, 2))  // None
```

`filter`：

```Scala
val x = divide(100, 2).filter(_ > 100)  // None
val x = divide(100, 2).filter(_ > 1)  // Some(50)
```

## for 表达式

`Option` 实现了 `map`、`flatMap` 和 `filter` 等函数，因此可用于 for 表达式：

```Scala
def divideString(a: String, b: String): Option[Double] =
  for {
    x ← parseDouble(a)
    y ← parseDouble(b)
    z ← divide(x, y)
  } yield z

divideString("11", "2")
```

## Scala `Option` vs Java `Optional`

熟悉 Java 的同学肯定知道，Java 8 引入了与 `Option` 类似的 `Optional`。

`Optional` 本身非常棒，非常推荐大家使用，但 Java 是一门古老的语言，Java 8 才引入的 `Optional` 注定无法向前兼容，例如 `Optional` 无法彻底融入现有的集合框架；而且 Java 也缺少类似模式匹配、for 表达式等可与 `Option` 完美配合的语言特性。因此 `Optional` 虽然能用，但相比 Scala `Option`，灵活性大大不如。

>通过诸如 [vavr](http://www.vavr.io/) 的函数式增强库，Java 也能获取模式匹配的能力。

---

参考：

* stackoverflow [Difference between Java Optional and Scala Option
](https://stackoverflow.com/questions/21714594/difference-between-java-optional-and-scala-option)