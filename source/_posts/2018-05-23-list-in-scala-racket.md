---
title: 天下 List 是一家
date: 2018-05-23 21:12:07
tags:
  - Scala
  - Racket
  - fp
categories: 技术
---

几乎所有编程语言都有 `List` 数据结构，尤其在函数式编程语言中，`List` 更是举足轻重，Lisp 甚至只用 `List` 就能一力降十会，可见其威力之无穷。

函数式编程语言中的 `List` 用法大同小异，本文比较 Racket 和 Scala 中 `List` 的惯用法，欣赏一下 `List` 的优美之处。

Racket 是动态函数式语言，Scala 是静态函数式语言，两者语言特性差别很大，但核心思想惊人的一致。

<!-- more -->

## 构造

>无论是 Racket，还是 Scala，列表默认都是不可变数据结构，定义完成后无法修改。

Racket 中的 `list` 是第二个元素为 `null` 的特殊 pair，`list` 递归定义为：

* 空列表 `null`
* 以空列表 `null` 为第二个元素的 `pair`

因此 `(define pr (cons 1 (cons #t "hi")))` 定义的是 `pair` 而非 `list`：

```Racket
(pair? pr) ; true
(list? pr) ; false
```

而 `(define lst (cons 1 (cons #t (cons "hi" null))))` 定义的是 `list` 而非 `pair`：

```Racket
(pair? lst) ; true
(list? lst) ; true
(list? null) ; true
```

Scala 中的 `List` 是通过 case 类定义的：

```Scala
sealed trait List[+A]

case class Cons[+A](head: A, tail: List[A]) extends List[A]
case object Nil                             extends List[Nothing]
```

Scala `List` 定义同样是递归的：

* 空列表 `Nil`
* 以 `List` 结尾的另一个 `List`

构造时使用 `::` 函数，最后一个参数必须是 `Nil`：

```Scala
val xs = 1 :: 2 :: Nil
```

* 这与 Standard ML 的列表定义非常类似。

无论 `cons` 还是 `::`，使用都并非特别方便，Racket 和 Scala 恰好都提供了一个更加方便的构造函数：

```Racket
(list #t "hi", 42)
```

* Racket 是动态语言，`list` 中元素类型可以不同

```Scala
val xs = List(1, 2, 3)
```

* Scala 是静态语言，`List` 中元素类型必须相同（实际可以创建类型为 `List[AnyVal]` 的 `List`，它可以包含任意类型的元素）

## 常见函数

下面手动实现几个常见的列表处理函数，读者将看到，虽然两种语言语法差别极大，但解决问题的核心思想却非常类似。

### `sum` 函数

Racket：

```Racket
(define (sum xs)
  (if (null? xs)
      0
      (+ (car xs) (sum (cdr xs)))))

(sum (list 1 2 3))  ; 6
```

Scala：

```Scala
def sum(xs: List[Int]): Int = xs match {
  case Nil      ⇒ 0
  case hd :: tl ⇒ hd + sum(tl)
}

sum(List(1, 2, 3))  // 6
```

### `append` 函数

Racket：

```Racket
(define (append xs ys)
  (if (null? xs)
      ys
      (cons (car xs) (append (cdr xs) ys))))

(append (list 1 2 3) (list 4 5))
```

Scala：

```Scala
def append[A](xs: List[A], ys: List[A]): List[A] = xs match {
  case Nil      ⇒ ys
  case hd :: tl ⇒ hd :: append(tl, ys)
}

append(List(1, 2, 3), List(4, 5))  // List(1, 2, 3, 4, 5)
```

### `map` 函数

Racket：

```Racket
(define (map f xs)
  (if (null? xs)
      null
      (cons (f (car xs)) (map f (cdr xs)))))

(map (lambda (x) (+ x 1)) (list 1 2 3 4))
```

Scala：

```Scala
def map[A, B](xs: List[A])(f: A ⇒ B): List[B] = xs match {
  case Nil      ⇒ Nil
  case hd :: tl ⇒ f(hd) :: map(tl)(f)
}

map(List(1, 2, 3))(_ + 1)  // List(2, 3, 4)
```

## 总结

前面几个函数都是通过递归实现的，从中可以看到：

* Racket 作为动态语言，无类型声明，非常简洁、灵活，但容易出错；
* Scala 作为静态语言，享受类型安全，但代码稍微繁琐；

Scala 的例子都是通过模式匹配实现的，Racket 不支持模式匹配，只能通过 `if` 表达式实现。

实际上，Scala `List` 处理一般使用 `foldLeft` 和 `foldRight`，前面的例子都可以用 `fold` 实现，相对模式匹配更加简洁：

```Scala
def sum(xs: List[Int]): Int =
  xs.foldLeft(0)(_ + _)

def append[A](xs: List[A], ys: List[A]): List[A] =
  xs.foldRight(ys)(_ :: _)

def map[A, B](xs: List[A])(f: A ⇒ B): List[B] =
  xs.foldRight(List.empty[B])((x, l) ⇒ f(x) :: l)
```