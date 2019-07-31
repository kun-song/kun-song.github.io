---
title: Scala | case 类
tags: case
categories: Scala
abbrlink: 23516
date: 2018-05-18 00:17:13
---

case 类适合对 **不可变数据**（immutable data）建模，主要用于模式匹配。

例如 JSON：

```Scala
abstract class JSON

case class JSeq(elements: List[JSON])           extends JSON
case class JObject(mappings: Map[String, JSON]) extends JSON
case class JNumber(value: Double)               extends JSON
case class JString(value: String)               extends JSON
case class JBool(value: Double)                 extends JSON
case object JNull                               extends JSON
```

<!-- more -->

还有链表：

```Scala
abstract class List[+A]
case class Cons[A](head: A, tail: List[A]) extends List[A]
case object Nil                            extends List[Nothing]
```

Scala 编译器会对 case 类做 **语法增强**：

* 工厂构造方法
* 开箱即用的 `toString`、`equals` 和 `hashCode` 方法
* 用于复制的 `copy` 方法
* 用于模式匹配的 `unapply` 方法

每一条增强并不起眼，但仅仅付出一个 `case` 关键字的代价，就能获取这么多增强，怎么看都稳赚不赔。

## 工厂方法

Scala 编译器会为 case 类自动生成 **伴生对象**，并在其中自动实现 `apply` 方法，有了 `apply` 方法创建 case 类实例时可省略 `new` 关键字：

```Scala
val number = JNumber(20)
```

创建 **嵌套** 的 case 类时，该特性可大大减少 **语法噪声**：

```Scala
val json = JObject(Map("name" → JString("Mike"), "age" → JNumber(20)))
```

若不用 case 类，则需要写作：

```Scala
val json = new JObject(Map("name" → new JString("Mike"), "age" → new JNumber(20)))
```

* 省略 `new` 大大增强了可读性

## 构造器参数 → 隐式 `val`

case 类构造器的参数被隐式声明为 `val`，因此将自动变成类中的 **`public` 字段**：

```Scala
val mappings = json.mappings
```

## `toString` & `equals` & `hashCode`

自动生成这些方法并不稀奇，但 Scala 编译器生成的这些方法开箱即用：

```Scala
// json: JObject = JObject(Map(name -> JString(Mike), age -> JNumber(20.0)))
val json = JObject(Map("name" → JString("Mike"), "age" → JNumber(20)))

// true
json == JObject(Map("name" → JString("Mike"), "age" → JNumber(20)))
```

* Scala 自动将 `==` 代理给 `equals`

## `copy` 方法

`copy` 方法用于创建与现有实例（大部分）**属性值相同** 的新实例：

```Scala
// list1: Cons[Int] = Cons(1,Cons(2,Nil))
val list1 = Cons(1, Cons(2, Nil))

// list2: Cons[Int] = Cons(1000,Cons(2,Nil))
val list2 = list1.copy(head = 1000)
```

* 仅将 `list1` 的 `head` 字段修改为 1000，其余保持一致

## `unapply` 方法

Scala 为 case 类自动生成 `unapply` 方法，以便用于模式匹配。

## 封闭类

将 case 类用于模式匹配时，为保证 **没有遗漏分支**，可将 case 类的 **共同父类** 声明为封闭类：

```Scala
sealed abstract class List[+A]  // 添加 sealed 关键字

case object Nil                             extends List[Nothing]
case class Cons[+A](head: A, tail: List[A]) extends List[A]
```

封闭类的 **所有子类** 必须与封闭类在 **同一源文件** 中定义，因此 Scala 编译器在 **编译时** 即可获取封闭类的所有子类信息，进而可以检查模式匹配分支的完备性。

## case 对象

若 case 类无任何构造参数，则可将其声明为 case 对象：

```Scala
case object Citizen {
  def firstName = "John"
  def lastName  = "Doe"
  def name = firstName + " " + lastName
}
```

case 对象与普通的 **单例对象** 区别如下：

首先，`case object` 将自动生成一个类，以及一个继承该类的对象：

```Scala
class Citizen { /* ... */ }
object Citizen extends Citizen { /* ... */ }
```

其次，case 对象依然具备上面介绍的 case 类的所有功能。

## 其他

case 类也是类，因此可以：

* 为 case 类手动添加方法、字段
* 修改编译器为 case 类自动生成的伴生对象

例如在 `List` 伴生对象中添加一个新的 `apply` 方法，用于便捷地创建列表：

```Scala
object List {
  def apply[A](xs: A*): List[A] =
    if (xs.isEmpty) Nil
    else Cons(xs.head, apply(xs.tail: _*))
}
```

有了新的 `apply` 方法（编译器已经为 `List` 生成过一个 `apply` 方法），创建列表方便很多：

```Scala
// xs: List[Int] = Cons(1,Cons(2,Cons(3,Cons(4,Nil))))
val xs = List(1, 2, 3, 4)
```

---

参考：

* [Tour of Scala](https://docs.scala-lang.org/tour/case-classes.html)
* [快学 Scala](https://book.douban.com/subject/27093751/)