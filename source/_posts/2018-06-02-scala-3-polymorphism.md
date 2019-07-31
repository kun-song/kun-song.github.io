---
title: 基本概念 | 多态知多少？
tags:
  - 多态
categories: Scala
date: 2018-06-02 11:40:57
---

多态（polymorphism）意为 many form, many shape，在计算机科学中，指 **单个接口** 可应用于 **多种类型** 的实体。

>这里的接口是广义的接口，既可以是函数，也可以是数据类型。

有 3 种常见的不同形式的多态：

* 参数多态（parametric polymorphism）
* 子类型多态（subtype polymorphism）
* 特设多态（ad hoc polymorphism）

>子类型 + 参数多态引出 variance 和 bounded quantification 的概念。

大多数编程语言具备其一或其二，而 Scala 三者皆有，下面以 Scala 和 Java 为例分别介绍三者。

<!-- more -->

## 参数多态

>参数多态最早由 ML 于 1975 年引入。

参数多态用参数表示值的类型，不限定具体类型，因此可用于 **无数** 类型，在保持静态类型安全的前提下，可大大增强语言的表现力。

参数多态既可用于 **函数**，又能用于 **数据类型**：

* 使用参数多态的函数称为 polymorphic function
* 使用参数多态的数据类型称为 polymorhic data type（如 `List`）

在 FP 和 OO 社区中，对参数多态称呼不同：

* FP 语言中参数多态随处可见，因此 **多态** 一般就是指参数多态；
* OO 语言
  + Java 和 C# 称之为泛型（generics）
  + C++ 和 D 称之为模板（template）

### Scala

>Scala 是 Rank-1 多态，另外还有 Rank-k 多态和 Rank-n 多态。

参数多态可用于数据类型：

```Scala
trait List[+A]
case class Cons[+A](head: A, tail: List[A]) extends List[A]
case object Nil                             extends List[Nothing]
```

`List` 和 `Cons` 均为多态数据类型（polymorphic data type），可用于任意类型：

```Scala
val xs = Cons(1, Cons(2, Cons(3, Nil)))
val ys = Cons("A", Cons("B", Cons("C", Nil)))
```

参数多态也可用于函数：

```Scala
object List {
  def length[A](xs: List[A]): Int = xs match {
    case Nil        ⇒ 0
    case Cons(_, t) ⇒ 1 + length(t)
  }

  def map[A, B](xs: List[A])(f: A ⇒ B): List[B] = xs match {
    case Nil        ⇒ Nil
    case Cons(h, t) ⇒ Cons(f(h), map(t)(f))
  }
}
```

`length` 和 `map` 可用于任意类型：

```Scala
List.length(xs)  // 3
List.map(xs)(_ + 1)  // Cons(2,Cons(3,Cons(4,Nil)))

List.length(ys)  // 3
List.map(ys)(_ * 2)  // Cons(AA,Cons(BB,Cons(CC,Nil)))
```

### Java

Java 称参数多态为泛型，集合框架泛型使用很普遍，前面的 `List` 在 Java 中定义如下：

```Java
public class List<E> {
    private Node<E> first;
    private Node<E> last;

    public void add(E ele) {
        Node<E> oldLast = last;
        last = new Node<>(oldLast, ele, null);

        if (oldLast == null) {
            first = last;
        } else {
            oldLast.next = last;
        }
    }

    private static class Node<E> {
        E item;
        Node<E> prev;
        Node<E> next;

        Node(Node<E> prev, E ele, Node<E> next) {
            this.item = ele;
            this.prev = prev;
            this.next = next;
        }
    }

    public int length() {
        int size = 0;
        Node<E> node = first;
        while (node != null) {
            size++;
            node = node.next;
        }
        return size;
    }

    @Override
    public String toString() {
        String result = "";
        Node<E> node = first;
        while (node != null) {
            result += node.item + ", ";
            node = node.next;
        }
        result = result.subSequence(0, result.lastIndexOf(",")).toString();

        return result;
    }
}
```

>由于 Java 比 Scala 啰嗦很多，这里我们需要定义 `add` 和 `toString` 函数。

我们为 `List` 定义一个泛型方法，用于便捷地创建 `List`：

```Java
public class Lists {
    public static <E> List<E> of (E... elements) {
        List<E> xs = new List<>();
        for (E e : elements) {
            xs.add(e);
        }

        return xs;
    }
}
```

`Lists.of` 是泛型方法，可用于不同类型的值，而 `List` 为泛型容器，可以保存不同类型的值：

```Java
List<Integer> xs = Lists.of(1, 2, 3);
List<String> ys = Lists.of("A", "B", "C");

System.out.println(ys);  // A, B, C
System.out.println(ys.length());  // 3

System.out.println(xs);  // 1, 2, 3
System.out.println(xs.length());  // 3
```

## 子类型多态

参数多态可用于 **任意类型**，而子类型多态的可用类型范围相对要小很多，而且仅应用于函数。

若函数 `f` 的参数类型为 `T`，且类型 `S` 为 `T` 的子类（写作 `S <: T`），根据 [里式替换原理](https://en.wikipedia.org/wiki/Liskov_substitution_principle) `f` 可接受 `S` 的实例作为实参，但实际使用 `S` 时，例如 `S.hi()` 时调用的是 `S` 而非 `T` 定义的 `hi` 方法。

无论在 Scala 还是 Java 中，子类型多态都是标配，大家早已习以为常，这里不过给它戴一个多态的帽子而已。

### Scala

假设有如下类型定义：

```Scala
trait Animal {
  def hi()
}

case class Cat(name: String) extends Animal {
  override def hi(): Unit = println(s"Hi, I'm a cat and my name is $name")
}
case class Dog(name: String) extends Animal {
  override def hi(): Unit = println(s"Hi, I'm a dog and my name is $name")
}
```

`Cat` 和 `Dog` 是 `Animal` 的子类，若想实现一个可同时用于 `Cat` 和 `Dog` 的函数，可以直接将参数类型定义为 `Animal`：

```Scala
def sayHi(animal: Animal): Unit = animal.hi()
```

也可用 Scala 类型上界简化 `sayHi` 的定义：

```Scala
def sayHi[A <: Animal](animal: A): Unit = animal.hi()
```

使用如下：

```Scala
sayHi(Cat("Mike"))  // Hi, I'm a cat and my name is Mike
sayHi(Dog("Bob"))  // Hi, I'm a dog and my name is Bob
```

### Java

Java 的子类型多态与 Scala 非常相似，略过不表。

## 特设多态

特设多态仅用于函数，指函数可应用于不同 **参数类型（或类型组合）**，根据参数类型不同，实际是调用的 **不同函数**。

大多数编程语言（Java）用 **函数重载** 或 **操作符重载** 实现特设多态，但 Scala 有所不同，下面详细说明。

### Scala

Scala 利用 **隐式转换** 或 **隐式参数** 实现特设多态，例如：

```Scala
trait CanPlus[A] {
  def plus(x: A, y: A): A
}

def plus[A](x: A, y: A)(implicit canPlus: CanPlus[A]): A =
  canPlus.plus(x, y)
```

与参数多态类似 `plus` 函数可用于任意类型 `A`，与参数多态不同的是，隐式作用域中必须有 `CanPlus[A]` 实例，因为 `plus` 实现依赖于 `CanPlus[A]` 实例。

为了让 `plus` 可用于 `String`，必须实现 `CanPlus[String]` 实例：

```Scala
implicit val canPlusString = new CanPlus[String] {
  override def plus(x: String, y: String): String = x + y
}

plus("Hello", "world")  // helloworld
```

为了让 `plus` 可用于 `Int`，必须实现 `CanPlus[Int]` 实例：

```Scala
implicit val canPlusInt = new CanPlus[Int] {
  override def plus(x: Int, y: Int): Int = x + y
}

plus(100, 100)  // 200
```

若无对应的 `CanPlus[A]` 实例，则 `plus` 将报错，例如 `plus(100.1, 100.2)` 报错如下：

```Scala
Error:(22, 72) could not find implicit value for parameter canPlus: A$A177.this.CanPlus[Double]
def get$$instance$$res2 = /* ###worksheet### generated $$end$$ */ plus(100.1, 100.2)
                                                                      ^
```

除隐式参数外，Scala 也可通过函数重载实现特设多态：

```Scala
object CanPlus {
  def plus(x: Int, y: Int): Int = x + y
  def plus(x: String, y: String): String = x + y
}
```

Scala 函数名可以是任意字符，可以实现 **类似** 操作符重载的效果（实际上 Scala 根本没有操作符的概念，一切皆函数）：

```Scala
object CanPlus {
  def +(x: Int, y: Int): Int = x + y
  def +(x: String, y: String): String = x + y
}
```

### Java

Java 不支持操作符重载，只能通过函数重载实现特设多态：

```Java
public class CanPlus {
    public static String plus(String s1, String s2) {
        return s1 + s2;
    }

    public static int plus(int x, int y) {
        return x + y;
    }
}
```

我们只为 `String` 和 `int` 实现了 `plus` 函数，因此只能通过这两种参数调用：

```Java
CanPlus.plus(1, 2);
CanPlus.plus("hello", "world);
```

## 总结

* 参数多态可用于函数、数据类型，而子类型多态、特设多态只能用于函数；
* 隐式参数实现的特设多态在 Scala 中非常常见，是实现 type class 的主要方式；
* Scala 多态要比 Java 简洁、强大很多；

---

参考：

* https://en.wikipedia.org/wiki/Polymorphism_(computer_science)
* [Herding Cats: What is polymorphism](http://eed3si9n.com/herding-cats/polymorphism.html)