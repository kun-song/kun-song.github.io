---
title: 读书 | Learn You a Haskell for Great Good
date: 2018-08-06 21:51:20
tags:
  - Haskell
  - 函数式
categories: 读书
---

作为一个 FP 粉，最终没能抵制住 Haskell 的诱惑，在 7 月份的空闲时间里把 [Learn You a Haskell for Great Good!](http://learnyouahaskell.com/chapters)（Haskell 趣学指南） 看了一遍。

本书被称为最好的 Haskell 入门书，讲解趣味，配图很赞，非常容易理解，基本做到了浅出，“缺点”也很明显：没有做到深入，鉴于它仅仅是入门书，这个缺点其实也不算缺点了。

我对函数式编程已经比较了解，书中对 FP 基本概念（高阶函数、递归、柯里化等）的讲解对我帮助不大，所谓“操千曲而后晓声，观千剑而后识器”，我已经在很多语言（Scala/ML/Racket）中一遍又一遍地实践它们了，不同语言写法不同，但思想都是相通的，这部分内容只能印证之前已有的知识，价值不大。

<!-- more -->

对我而言，本书价值最大的部分是对 typeclass 的讨论，虽然 Scala 也可以实现 typeclass，并且社区也有诸如 Scalaz/Cats 的库，但 typeclass 毕竟不是 Scala 的内置特性，用 Scala 学习 typeclass 容易雾里看花，抓不到本质。

而 Haskell 对 typeclass 的支持是语言级别的，非常简单：

* `class` 声明 typeclass
* `instance` 实现 typeclass instances

比如定义 `Monad` typeclass：

```Haskell
class Monad m where
  return :: a -> m a
  (>>=) :: m a -> (a -> m b) -> m b
  (>>) :: m a -> m b -> m b
  x >> y = x >>= \_ -> y

  fail :: String -> m a
  fail msg = error msg
```

然后将 `Maybe` 实现为 `Monad` 实例：

```Haskell
instance Monad Maybe where
  return = Just
  Nothing >>= _ = Nothing
  Just x >>= f  = f x
```

之后就可以将 `>>=` 函数用于 `Maybe`：

```Haskell
   return 10 :: Maybe Int
=> Just 10
   Just 10 >>= \x -> Just $ x + 5
=> Just 15
   Nothing >>= \x -> return $ x + 5
=> Nothing
```

作为对比，用 Scala 实现同样的功能，首先定义 typeclass：

```Scala
trait Monad[M[_]] {
  def pure[A](a: A): M[A]
  def flatMap[A, B](ma: M[A])(f: A ⇒ M[B]): M[B]
}

object Monad {
  def apply[M[_]: Monad]: Monad[M] = implicitly[Monad[M]]
}
```

然后为 `flatMap` 实现便捷语法：

```Scala
object MonadSyntax {
  implicit class MonadOps[M[_]: Monad, A](ma: M[A]) {
    def >>=[B](f: A ⇒ M[B]): M[B] = Monad[M].flatMap(ma)(f)
  }
}
```

最后为 `Option` 实现 `Monad` 实例：

```Scala
object MonadInstances {
  implicit val optionMonad: Monad[Option] = new Monad[Option] {
    override def pure[A](a: A): Option[A] = Some(a)
    override def flatMap[A, B](ma: Option[A])(f: A ⇒ Option[B]): Option[B] = ma.flatMap(f)
  }
}
```

使用：

```Scala
object Test extends App {
  import MonadInstances._
  import MonadSyntax._

  val x = Option(111)
  val f = (x: Int) ⇒ Option(x * 6)
  val g = (x: Int) ⇒ Option(x + 1)

  val y = x >>= f >>= g

  println(y)  // Some(667)
}
```

可以看到用 Scala 实现 typeclass 比用 Haskell 多出以下内容：

* Scala 没有提供创建 typeclass 的关键字，导致 typeclass 完全淹没在 `trait`、`object`、`implicit` 转换等语法噪音中，光找到到 typeclass 就很不容易了
* 隐式转换
* `implicit` 解析规则比较复杂，又得牺牲一些脑细胞

与 Haskell 一对照就会发现，这些都属于实现细节，与 typeclass 概念本身毫无关系，从这个意义上讲，Haskell 可以让大家更容易学会 typeclass。

最后，学习语言特性好无聊啊，有空研究下语言实现 :)

---

另外，读书做笔记是个好习惯：[本书笔记](https://github.com/satansk/learn-you-a-haskell)。
