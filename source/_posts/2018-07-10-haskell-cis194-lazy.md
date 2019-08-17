---
title: CIS 194 | Lazy
date: 2018-07-10 22:21:34
tags:
  - lazy
categories: 公开课
---

这里的 lazy 指 lazy evaluation，是一种编程语言的求值策略，与之对立的是 strict evaluation，更多关于求值策略的内容可以参考 {% post_link 2018-05-17-call-by-value-call-by-name 传值调用 vs 传名调用 %}。

不同语言采用了不同的求值策略：

* 大多数编程语言采用 strict 求值策略，如 Java；
* 部分语言同时提供 strict 和 lazy 两种求值策略，如 Scala；
* 少数语言更加极端，全面拥抱 lazy，例如 Haskell、Miranda 和 Clean 等。

Haskell 选择 lazy 只是一种语言设计上的决策而已，既有好处，也有缺点。

<!-- more -->

大部分同学由于只接触过 strict 策略，所以对 lazy 这个语言特性带来的一些结果不甚了了，实际上借助 lazy 可以实现很多有意思的功能，例如：

* 自定义控制结构（control structures）
* 无限数据结构（infinite data stucture)

>熟悉 Scala 的同学可以发现，Scala 可轻易实现类似功能，毕竟 Scala 也可以选择 lazy 求值策略。

但全面采用 lazy 会导致代码的运行效率难以推断，只能一行一行研读代码之后，才能对 Haskell 的计算过程有所了解，在某些场景下这简直是噩梦。

## 自定义控制结构

在 Java 等语言中，`&&` 和 `||` 操作符是短路求值的，它们是语言 **内置** 特性，在 Haskell 中，可以用函数轻易实现 `&&`：

```Haskell
(&&) :: Bool -> Bool -> Bool
True && x  = x
False && _ = False
```

上面定义的 `&&` 函数与 Java 中的内置 `&&` 操作符完全相同：

```Haskell
-- False
False &&! (34^9784346 > 34987345)
False &&! (head [] == 'x')
```

作为对比，Scala 也能实现类似功能，不过要繁琐一些：

```Haskell
def and(x: Boolean, y: => Boolean): Boolean =
  if (x) y else false
```

再看一个例子，`if` 在 Java 中也是内置的，而 Haskell 也可以用函数轻易实现之：

```Haskell
if :: Bool -> a -> a -> a
if True x _  = x
if False _ y = y
```

## 无限数据结构

无限数据结构在《Scala 函数式编程》第 5 章有非常好的讲解，这里仅仅列出一些 Haskell 代码片段。

自然数列表：

```Haskell
nats :: [Integer]
nats = 0 : map (+ 1) nats
```

CIS 194 第 6 周作业实现了一个“显式”的无限列表：

```Haskell
data Stream a = Cons a (Stream a)
```

* 实际上，Haskell 中的 list 既可以是有限的，也可以是无限的，这里用 `Stream` 是为了“显式”表明它只能是无限列表

为其实现 `Functor` 实例：

```Haskell
instance Functor Stream where
    fmap f (Cons x s) = Cons (f x) $ fmap f s
```

所以前面的 `nats` 也可以用 `Stream` 重新定义：

```Haskell
nats :: Stream Integer
nats = Cons 0 $ (+ 1) <$> nats
```

还可以将 `Stream` 转换为普通 list：

```Haskell
streamToList :: Stream a -> [a]
streamToList (Cons x s) = x : streamToList s
```

还有一些便捷函数：

```Haskell
sRepeat :: a -> Stream a
sRepeat x = Cons x $ sRepeat x

sIterate :: (a -> a) -> a -> Stream a
sIterate f x = Cons x $ sIterate f $ f x

sInterleave :: Stream a -> Stream a -> Stream a
sInterleave (Cons x s1) s2 = Cons x $ sInterleave s2 s1

sTake :: Int -> Stream a -> [a]
sTake n s = take n $ streamToList s
```

## 求值时机

即使 lazy 代码，早晚也得计算，那么 Haskell 何时对各个表达式求值呢？

一句话解答：

* Expressons are only evaluated when pattern-matched
* ... only as far as necessary for the match to **proceed**, and no further!

即表达式只有 **被模式匹配** 时才会被计算，而且只要满足模式匹配，就不再继续向前计算。

例如：

```Haskell
f1 :: Maybe a -> [Maybe a]
f1 m = [m, m]

f2 :: Maybe a -> [a]
f2 Nothing  = []
f2 (Just x) = [x]
```

虽然 `f1` 函数体也用到了参数值，但无需了解任何 `m` 的性质即可完成计算（`f1 e` 不依赖 shape of e），因此 `f1` 不需要计算参数 `m`。

相对而言，`f2` 必须了解参数的 **结构** 才能完成计算（`Nothing` or `Just a`），因此 `f2` 需要计算参数；

但 `f2 (safeHead [3^100 + 10, 44])` 调用中，只需要计算 `safeHead [3^100 + 10, 44]` 得到 `Just 3^100 + 10`，不会继续计算 `3^100 + 10`，因为计算到这已经“足够”支持模式匹配的进行了。
