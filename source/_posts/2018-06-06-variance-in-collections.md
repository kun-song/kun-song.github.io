---
title: Scala | 集合中的型变
date: 2018-06-06 23:49:17
tags:
  - Scala
  - 类型系统
  - 型变
categories: 技术
---

在 {% post_link scala-variance Scala | 型变 %} 一文中我简单介绍了 Scala 中的协变、逆变和不变，Scala 的集合类型一般是协变 or 不变，本文将简单介绍不同集合选择协变 or 不变的原因，进一步加深对 Scala 型变（variance）的理解。

一言以蔽之：

* 不可变集合（immutable collections）的类型应为协变（covariant）
* 可变集合（mutable collections）的类型应为不变（invariant）

下面介绍这样设计的原因。

<!-- more --->

## 不可变集合

不可变集合的方法会返回一个 **新集合**，原来的集合不受任何影响，根据该特点，可将其定义为协变，以获取更加 **符合直觉** 的子类型关系（若 `A` 为 `B` 的子类，则 `List[A]` 是 `List[B]` 的子类）：

```Scala
trait List[+T] {
  def add[U >: T](other: U): List[U]
}
```

`add` 接受元素类型的 **父类** 作为参数，并将返回的集合 **降级**，即将 `List[T]` 降低为 `List[U]`，结果是一个更加“宽泛”的集合。

假设有如下类定义：

```Scala
abstract class Animal {
  def name: String
}
case class Cat(name: String) extends Animal
case class Dog(name: String) extends Animal
```

以及方法：

```Scala
def printAnimalNames(animals: List[Animal]): Unit = {
  animals.foreach { animal =>
    println(animal.name)
  }
}
```

因为 `List` 为协变，所有 `List[Dog]` 和 `List[Cat]` 作为 `List[Animal]` 的子类都可以用于 `printAnimalNames` 方法，非常灵活：

```Scala
val cats: List[Cat] = List(Cat("Whiskers"), Cat("Tom"))
val dogs: List[Dog] = List(Dog("Fido"), Dog("Rex"))

printAnimalNames(cats)
printAnimalNames(dogs)
```

## 可变集合

协变很有用，那是否可以把可变集合定义为协变呢？

答案是否定的，因为协变会导致可变集合损失 **类型安全**，是 **无效** 的。

假如把可变集合 `HashSet` 定义为协变：

```Scala
trait HashSet[+T] {
  def add[U >: T](item: U)
}
```

* `add` 原地修改集合

现在有一个 `HashSet[Dog]`：

```Scala
val dogs: HashSet[Dog]
```

因为我们把 `HashSet` 定义为协变，因此 `HashSet[Animal]` 是 `HashSet[Dog]` 的父类，因此可以把 `HashSet[Dog]` 视为 `HashSet[Animal]`：

```Scala
val animals: HashSet[Animal] = dogs
```

现在 `animals` 是 `HashSet[Animal]`，自然可以往里添加一个 `Cat`：

```Scala
mammals.add(Cat("Mike"))
```

到此为止，一切都是合法的。

问题是 `animals` 实际指向 `dogs`，而 `dogs` 实际类型依然是 `HashSet[Dog]`，将其视为 `HashSet[Animal]` 不过是一种假象而已（协变导致），但现状是 `dogs` 中包含了 `Cat`，明显失去了类型安全。

因此，可变集合只能是不变。

---

参考：

* [Effective Scala](http://twitter.github.io/effectivescala/index-cn.html#%E7%B1%BB%E5%9E%8B%E5%92%8C%E6%B3%9B%E5%9E%8B-%E5%8F%98%E5%9E%8B)