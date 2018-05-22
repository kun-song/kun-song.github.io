---
title: Scala | 型变
tags:
  - Scala
  - 类型系统
categories: 技术
abbrlink: 19302
date: 2018-05-15 00:39:34
---

Scala 是一门多范式编程语言，融合了面向对象与函数式编程两种范式。

Scala 通过类型参数实现了参数多态，而作为 OOP 语言，又必然允许类的继承，当 **参数多态** 遇上 **类继承** 时，出现了一个有趣的问题：

若类型 `S` 是 `T` 的子类，那么 `List[S]` 与 `List[T]` 是什么关系呢？（`List` 泛指泛型容器）

<!-- more -->

Scala 类型系统必须能够解释该问题，在回答之前，我们先要搞清楚类继承到底有什么意义，其意义可以用 [里式替换原理](https://en.wikipedia.org/wiki/Liskov_substitution_principle) 说明：

>If S is a subtype of T, then objects of type T may be replaced with objects of type S.

通过 subtype 还可以实现 [subtype polymorphism](https://en.wikipedia.org/wiki/Subtyping)，当然这扯的有点远了。

回到前面的问题，`List[S]` 和 `List[T]` 到底是不是父子类关系呢？如果是，则根据里式替换原理，使用它们更加灵活。

为解决该问题，Scala 提供了 3 种型变标记，以表达 `List[S]` 与 `List[T]` 之间的关系：

|               |                meaning           |        notion      |
|       :---:   |                :----:           |        :----:     |
| invariant     | `List[S]` 与 `List[T]` 无任何关系 |       `[T]`       |
| covariant     | `List[S]` 是 `List[T]` 的子类     |       `[+T]`      |
| contravariant | `List[T]` 与 `List[S]` 的子类     |       `[-T]`      |

## 协变

Scala 的不可变集合，基本都定义为协变，例如 `List`：

```Scala
sealed abstract class List[+A] {
  def ::[B >: A] (x:B) : List[B]
}
```

使用场景：

```Scala
abstract class Animal(name: String)
case class Bird(name: String) extends Animal(name)
case class Duck(name: String) extends Animal(name)

val birds = new Bird("A") :: new Bird("B") :: Nil
val ducks = new Duck("a") :: new Duck("b") :: Nil

def toString(animals: List[Animal]): Unit = println(animals.map(_.toString).mkString(", "))

toString(birds)
toString(ducks)
```

`toString` 参数为 `List[Animal]`，而：

* `Duck` 和 `Bird` 是 `Animal` 的子类，`List` 是协变容器
* 因此 `List[Bird]` 和 `List[Duck]` 是 `List[Animal]` 的子类

所以 `List[Bird]` 和 `List[Duck]` 都可以调用 `toString` 函数（里式替换原理）。

## 逆变

协变比较容易理解，但逆变有点反直觉，其实函数定义就用到了逆变：

```Scala
trait PartialFunction[-A, +B] extends (A => B)
```

暂时忽略协变的返回类型，`PartialFunction` 的入参是逆变的，这有什么意义呢？

假设有如下类定义：

```Scala
class Animal(val name: String)
class Bird(name: String) extends Animal(name)
class Duck(name: String) extends Bird(name)
```

假设现在需要实现一个函数，以获取 `Bird` 的名字：

```Scala
val getBirdName: Bird ⇒ String = ???
```

但 `Animal` 标准库中已经有获取名字的函数了：

```Scala
val getName: Animal ⇒ String = _.name
```

`getName` 类型为 `Animal => String`，即 `Function[Animal, String]`，`getBirdName` 类型为 `Function[Bird, String]`，因为：

* `Function[-A, +B]` 入参是逆变的；
* `Bird` 是 `Animal` 的子类；

因此 `Function[Animal, String]` 是 `Function[Bird, String]` 的子类，即 `getName` 是 `getBirdName` 的子类，所以可以用 `getName` 替换 `getBirdName`：

```Scala
val getBirdName: Bird ⇒ String = getName
```

>`Function1[-A, +B]` 入参为协变是否合理？
>
>非常合理，假如第一个参数是协变，即 `Function1[+A, +B]`，则：
>* `Bird => String` 是 `Animal => String` 的子类
>
>因此可以用 `getBirdName` 替换 `getName`，**但是** `getName` 可以用于任意 `Animal`，虽然语法上 `getBirdName` 替换掉了 `getName`，但是 `getBirdName` 并不能用于任意 `Animal`，因此早晚会报错：
>
>若有 `Human extends Animal`，则 `getBirdName` 无法用于 `Human` 实例。

`Function1` 的返回类型为协变，这也非常合理，假设 `f` 返回类型为 `Bird`，`g` 返回类型为 `Duck`，自然可以用 `g` 替换 `f`，毕竟 `Duck` 可以替换 `Bird`：

```Scala
val f: () => Bird = () => new Duck("duck")
```

## 不变

如果既没有 `+` 也没有 `-`，则默认为不变，即 `List[S]` 和 `List[T]` 没有任何关系。

## Java ？

Java 也支持 parametric polymorphism，因此也存在同样的问题，Java 也有自己的一套解决方式，留待另一篇博文解释。