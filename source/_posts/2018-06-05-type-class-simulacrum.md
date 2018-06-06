---
title: 【译】Type class in Scala
date: 2018-06-05 00:49:52
tags:
  - Scala
  - fp
  - Simulacrum
  - Type Class
categories: 技术
---

>本文写的非常好，忍不住翻译一下，原文链接 [在此](https://blog.scalac.io/2017/04/19/typeclasses-in-scala.html)。

Type class 是一个强大、灵活的概念，Scala 通过 type class 实现特设多态（ad-hoc polymorphism）。

在 Scala 中，type class 并非第一等公民，但仅通过一些 Scala 内置的语言特性就能实现它，这导致两个不那么美好的结果：

* 不太容易在代码中发现 type class
* 大家对如何“正确”地实现 type class 有点迷惑

本文总结 type class 的概念、工作原理，以及如何在 Scala 中实现 type class。

<!-- more -->

## 概念

type class 最初（1988）是作为一种 **新** 的实现特设多态的方法由 Haskell 引入的（见论文 How to make ad-hoc polymorphism less ad hoc），Haskell 从语言层面支持 type class，将其作为 Hindley-Milner 类型系统的扩展。

>type class is a class(group) of types.

这组类型必须满足：

* 符合 `trait` 中定义的合约（contract）
* 可以在 **不修改代码** 的条件下添加 type class 代表的功能（`trait` 及其实现）

有人会说直接混入 `trait` 岂不是更简单？

混入 `trait` 自然可满足第一条，但第二条无法满足。

与 Haskell 不同，Scala 并未提供实现 type class 的特定语法，仅仅用 Scala 的普通语言特性即可实现它，因此菜鸟们不太容易发现代码中隐藏的 type class，典型的 type class 实现必然会使用一些语法糖，进一步提升了发现它们的难度。

因此，让我们一步一步地实现一个 type class，并理解它的原理。

## 实现

接下来我们实现一个 type class，它可以获取给定类型的字符串表示，所有值都可以使用该 type class，类似 `toString`。

### 第一步

Scala 使用 `trait` 表示 type class 本身：

```Scala
trait Show[A] {
  def show(a: A): String
} 
```

我们要添加的功能为 `show`，该功能可用于 **所有类型**，但却定义在任意类型 **之外** 的特质上。

为 `Int` 实现 `show` 功能：

```Scala
object Show {
  val intShow: Show[Int] =
    new Show[Int] {
      override def show(a: Int): String = s"Int: $a"
    }
}
```

`intShow` 是一个 `Show[Int]` 实例，根据惯例，我们把它放到 `Show` 伴生对象中。

现在可以使用 `intShow` 了，虽然非常笨重：

```Scala
import Show._

intShow.show(10)  // "Int: 10"
```

### 第二步

为 **避免** 直接调用 `intShow`，我们在 `Show` 伴生对象中实现一个同名的 `show` 方法：

```Scala
object Show {
  def show[A](a: A)(implicit sh: Show[A]): String = sh.show(a)

  implicit val intShow: Show[Int] =
    new Show[Int] {
      override def show(a: Int): String = s"Int: $a"
    }
}
```

`show` 方法第二个参数为 `implicit`，且 `Show` 对象中恰好有一个 `implicit Show[Int]` 实例，因此当有如下调用时，Scala 编译器可自动为 `show` 补全第二个参数：

```Scala
import Show._

show(10)  // 编译器自动补全为 show(10)(intShow)
```

到此为止，我们已经实现了一个 **基本** 的 type class，虽然还有很大的优化空间，但已经五脏俱全：

* type class 本身
  + 特质 `Show`
* type class instaces
  + 为 `Int` 实现了一个 type class 实例
* 对外接口
  + 具备 `implicit` 参数的 `Show.show` 方法
  + 隐式的 type class instance

Scala 为 `implicit` 参数提供了 context bound 语法，更加便捷：

```Scala
def show[A: Show](a: A): String = implicitly[Show[A]].show(a)
```

>* context bound 主要就是用于 type class
>* context bound + implicitly 形影不离

type class 还有一个惯用法：一般不用 `implicitly` 函数，而是通过伴生对象中的 `apply` 方法获取同样效果。

为 `Show` 定义 `apply` 方法，该方法仅有一个 `implicit` 参数：

```Scala
def apply[A](implicit sh: Show[A]): Show[A] = sh
```

* 通过编译器捕获到合适的参数后，`apply` 直接返回该参数

将 `show` 方法中的 `implicitly` 替换为 `Show.apply`：

```Scala
def show[A: Show](a: A): String = Show.apply[A].show(a)
```

`apply` 可以简化掉：

```Scala
def show[A: Show](a: A): String = Show[A].show(a)
```

### 第三步

`show(10)` 固然不错，但若能 `10 show` 岂不是更好？

我们通过隐式转换实现该特性：

```Scala
implicit class ShowOps[A: Show](a: A) {
  def show: String = Show[A].show(a)
}
```

>惯用法：该隐式转换类一般称为 TypeclassName + Ops

有了 `Ops` 类，我们可以这样做：

```Scala
10 show
```

为避免运行时开销，可以将 `ShowOps` 定义为值类型，并将 type class 约束放到 `show` 方法中：

```Scala
implicit class ShowOps[A](val a: A) extends AnyVal {
  def show(implicit sh: Show[A]): String = sh.show(a)
}
```

做了这么多工作，现在 `Show` 伴生对象内容如下：

```Scala
object Show {
  def apply[A](implicit sh: Show[A]): Show[A] = sh

  def show[A: Show](a: A): String = Show[A].show(a)

  implicit class ShowOps[A: Show](a: A) {
    def show: String = Show[A].show(a)
  }

  implicit val intShow: Show[Int] =
    new Show[Int] {
      override def show(a: Int): String = s"Int: $a"
    }
}
```

现在为我们的 type class 再添加一个实例：

```Scala
implicit val stringShow: Show[String] =
  new Show[String] {
    override def show(a: String): String = s"String: $a"
  }
```

`stringShow` 实现与 `intShow` 非常类似，可以抽象出一个 type class instances 构造器：

```Scala
def instance[A](f: A ⇒ String): Show[A] =
  new Show[A] {
    override def show(a: A): String = f(a)
  }

implicit val intShow: Show[Int] =
  instance(int ⇒ s"Int: $int")

implicit val stringShow: Show[String] =
  instance(s ⇒ s"String: $s")
```

为 type class 添加 instance 辅助函数非常常见，但 Scala 2.12 以后可以直接使用 Single Abstract Method，更加简洁：

```Scala
implicit val intShow: Show[Int] =
  int ⇒ s"Int: $int"

implicit val stringShow: Show[String] =
  s ⇒ s"String: $s"
```

### 更多

到目前为止，我们实现一个简单的 type class，提供了两种 `show` 函数的调用方式（`show(10` 和 `10 show`)，并为 `Int` 和 `String` 两个类型实现了 type class instances，完整代码如下：

```Scala
trait Show[A] {
  def show(a: A): String
}

object Show {
  def apply[A](implicit sh: Show[A]): Show[A] = sh

  def show[A: Show](a: A): String = Show[A].show(a)

  implicit class ShowOps[A: Show](a: A) {
    def show: String = Show[A].show(a)
  }

  implicit val intShow: Show[Int] = int ⇒ s"Int: $int"

  implicit val stringShow: Show[String] = s ⇒ s"String: $s"
}
```

若使用 `import Show._` 导入全部内容，则用户无法自己实现一些 type class instance，覆盖掉默认的实例，因为全部实例都被引入当前作用域了，再实现一个同类型的实例会引起歧义。

因此可以将 `show` 方法与 `ShowOps` 放到另一个单独的对象 `ops` 中：

```Scala
object Show {
  def apply[A](implicit sh: Show[A]): Show[A] = sh

  object ops {
    def show[A: Show](a: A): String = Show[A].show(a)

    implicit class ShowOps[A: Show](a: A) {
      def show: String = Show[A].show(a)
    }
  }

  implicit val intShow: Show[Int] = int ⇒ s"Int: $int"

  implicit val stringShow: Show[String] = s ⇒ s"String: $s"
}
```

如此一来，若用户不想使用默认的 type class instances，可以只导入 type class 和对外接口：

```Scala
import xx.Show
import xx.Show.ops._
```

这样导入的结果是：

* 用户可以在 category 1 implicits 中自定义 type class 实例
* 默认实例在 category 2 implicits 中

因此若用户提供了自定义的 type class 实例，则可以覆盖 category 2 implicits 中的默认实例，若没有提供自定义实例，则依旧可以使用 category 2 implicits 中的实例。

到目前为止，我们的 type class 基本成型。

## 自定义类型

### 自定义类型

type class 作者一般会为常见类型提供 type class instances，但我们的自定义类型一般无法直接使用 type class。

当然，我们可以为自定义类型自行实现 type class 实例，可以在任意位置放置这些新实例，若有 type class 源码，放在其伴生对象中也是极好的。

若有自定义类型 `Foo`：

```Scala
case class Foo(foo: Int)
```

我们可以为其实现 type class 实例：

```Scala
import xx.Show
import xx.Show.ops._

implicit val fooShow: Show[Foo] =
  foo ⇒ s"case class Foo(foo: ${foo.foo})"

Foo(100) show  // case class Foo(foo: 100)
```

### Shapeless

本段有点偏题，但我（作者）认为值得一提。

Scala 实现 type class 的方式（`implicit`）让我们可以借助 Shapeless 为自定义类型 **自动** 生成 type class 实例。

例如我们可以为 **所有** `case` 类（实际上，是所有 product type）定义 `show` 方法。这需要我们为基本类型定义 type class 实例，以及确定如何为 product type 定义 `show` 方法，虽然不那么简单，但却省却为每个新类型定义实例的繁琐。

## Simulacrum

[Simulacrum](https://github.com/mpilquist/simulacrum) 通过宏为 type class 添加便捷语法，是否使用取决于个人判断。

若使用 Simulacrum，则可以一眼找出代码中所有的 type class，并且可省去很多样板代码。

另一方面，使用 `@typeclass`（Simulacrum 主要注解）则意味着需要依赖 macro paradise 编译器插件。

使用 Simulacrum 重写我们的 `Show` type class：

```Scala
import simulacrum._

@typeclass trait Show[A] {
  def show(a: A): String
}
```

有了 Simulacrum，type class 定义变得非常简洁，我们在其伴生对象中添加 `Show[String]` 实例：

```Scala
object Show {
  implicit val stringShow: Show[String] = s ⇒ s"String: $s"
}
```

使用方式与之前完全一致：

```Scala
import xx.Show.ops._

"Hello" show
```

>Simulacrum 会为 `Show` 自动生成 `ops` 对象，与前面自定义的基本一致。

## implicits

Scala 使用 implicits（隐式转换 + 隐式参数）实现 type class，因此 type class 既能享受 implicits 的优势，也必须承担 implicits 的代价。

可以为类型 `A` 定义 **多个** type class 实例，Scala 编译器将按 **优先级** 进行 implicit 解析，找到与当前位置最接近的实例，而 Haskell 只允许定义 **一个** type class 实例。

我们既可以依赖 Scala 编译器自动为 `implicit` 参数提供值，也可以手动传递 `implicit` 参数值，手动传递在某些场景中很有用。

为支持手动传递 `implicit` 参数，我们需要为 `ShowOps` 类添加一个新方法 `showExp`：

```Scala
implicit class ShowOps[A: Show](a: A) {
  def show: String = Show[A].show(a)

  def showExp(implicit sh: Show[A]): String = sh.show(a)
}
```

>注意：`ShowOps.show` 是无参方法，当然无法手动传递参数。

`showExp` 支持两种调用方式：

```Scala
val hipsterString: Show[String] =
  s ⇒ s"hipsterString: $s"

"Hello".showExp  // String: Hello
"Hello".showExp(hipsterString)  // hipsterString: Hello
```

* `"Hello".showExp` 将使用 category 2 implicits 中定义的 `Show[String]` 实例（`stringShow`）做实参
* `"hello".showExp(hipsterString)` 强制使用 `hipsterString`

通过添加 `showExp` 可以支持两种不同调用方式，实际上更常见的做法是不添加 `showExp` 函数，而是在 category 1 implicit 中添加一个新的隐式 type class 实例，利用 implicit 优先级，覆盖 category 2 implicit 中默认 type class 实例：

```Scala
"Hello".show  // String: Hello

implicit val hipsterString: Show[String] =
    s ⇒ s"hipsterString: $s"

"Hello".show // hipsterString: Hello
```

* 第一个 `"Hello".show` 将使用 `Show` 伴生对象定义的默认 type class 实例
* 第二个 `"Hello".show` 将使用 `hipsterString`

## implicit 弊端

使用 implicit 实现 type class 也会带来一些弊端。

首先，不能为相同类型 `T` 定义 **多个优先级相同** 的 type class 实例。

这听起来似乎没什么大不了的，但实践中会导致一些问题。例如当使用 Cats/Scalaz 时很容易出现关于歧义 implicit 的编译错误，该问题一般与 type class 的实现方式相关，可能出现多个不同 implicit 实例同时满足编译器的查找条件，更多讨论在 [这里](https://typelevel.org/blog/2016/09/30/subtype-typeclasses.html)。

且错误提示信息容易误导，例如下面调用中，编译器根本不知道 `show` 是个 type class 定义的行为：

```Scala
true.show
```

因此编译器只能提示：

```Scala
value show is not a member of Boolean.
```

## 开源示例

开源代码是学习 type class 的好去处，例如：

* [Cats](https://github.com/typelevel/cats)
  + Cats 以 type class 作为首要建模方式
  + Cats 也使用了 simulacrum
* [Shapeless](https://github.com/milessabin/shapeless)
  + Shapeless 也严重依赖 type class
  + `HList`

## type class 展望

有很多关于如何为 type class 添加语法的讨论：

* [Think about type class/implicits encoding](https://github.com/lampepfl/dotty/issues/2029)
* [Simulacrum](https://github.com/mpilquist/simulacrum)
* [Dandy](https://github.com/maxaf/dandy)

也有一些关于 type class 一致性的讨论：

* [Allow Typeclasses to Declare Thenselves Coherent](https://github.com/lampepfl/dotty/issues/2047)

## 总结

从概念上讲，type class 非常简单；但实现时，却有很多 corner cases。

type class 一般用于类库设计，业务应用中较少使用，但了解 type class 总是好的。
