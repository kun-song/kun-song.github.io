---
title: A tiny interpreter in Haskell
date: 2018-06-24 21:20:09
tags:
  - fp
  - 解释器
  - CIS 194
  - Haskell
  - 环境
  - 语法糖
categories: 公开课
---

最近在跟宾夕法尼亚大学的 Haskell 课程（见 {% post_link 2018-06-15-haskell-begin-cis-194 正式入坑 Haskell %}），这周末两天完成了第二周、第三周的学习内容。

第三周作业是实现 SImPL 语言的解释器，题目不难，所有函数基于已生成的 AST 进行操作。

通过该练习，可以加深对以下概念理解：

* 状态
* 语法糖
* 表达式 vs 语句
* 求值

下面简单介绍下解析器的实现过程。

<!-- more -->

## SImPL 语言介绍

>详细信息参考 [作业说明](http://www.seas.upenn.edu/~cis194/spring15/hw/03-ADTs.pdf)。

首先，SImPL 是一个假想语言，只支持 `Int` 类型，支持变量，总共有 7 种语句：

* assignment
* increment
* if
* while
* for
* sequence
* skip

虽然 SImPL 非常简单，但已经可以实现很多复杂的计算了，例如：

```Haskell
// B^E
Out := 1;
 for (I := 0; I < E; I++) {
Out := Out * B }

// 阶乘
for (Out := 1; In > 0; In := In - 1) {
  Out := In * Out
}

// 平方根
B := 0;
  while (A >= B * B) {
    B++
  };
B := B - 1

// 斐波那契数列
F0 := 1;
F1 := 1;
if (In == 0) {
  Out := F0
} else {
  if (In == 1) {
    Out := F1
  } else {
    for (C := 2; C <= In; C++) {
      T  := F0 + F1;
      F0 := F1;
      F1 := T;
      Out := T
    }
  }
}
```

## 全局状态

全局状态有很多种实现方式，比如数组、列表等等，在 Haskell 中，我们用一个函数表示全局状态，给定变量名字，`State` 将返回名字的值：

```Haskell
type State = String -> Int
```

可以通过 `extend` 函数扩展状态：

```Haskell
extend :: State -> String -> Int -> State
extend st name value = aux
  where aux x
          | x == name = value
          | otherwise = st x

empty :: State
empty _ = 0
```

* `empty` 是初始状态

## 表达式求值

表达式无法凭空计算，需要先根据 `State` 查询变量的值，具体的求值过程是基于模式匹配的，非常简单：

```Haskell
evalE :: State -> Expression -> Int
evalE st (Var name)         = st name
evalE st (Val v)            = v
evalE st (Op e1 Plus e2)    = evalE st e1 + evalE st e2
evalE st (Op e1 Minus e2)   = evalE st e1 - evalE st e2
evalE st (Op e1 Times e2)   = evalE st e1 * evalE st e2
evalE st (Op e1 Divide e2)  = evalE st e1 `div` evalE st e2
evalE st (Op e1 Gt e2)      = fromEnum (evalE st e1 > evalE st e2)
evalE st (Op e1 Ge e2)      = fromEnum (evalE st e1 >= evalE st e2)
evalE st (Op e1 Lt e2)      = fromEnum (evalE st e1 < evalE st e2)
evalE st (Op e1 Le e2)      = fromEnum (evalE st e1 <= evalE st e2)
evalE st (Op e1 Eql e2)     = fromEnum (evalE st e1 == evalE st e2)
```

## 语法脱糖

SImPL 的 7 种语句中，有两个是语法糖，是为方便用户设计的：

* increment 是 assignment 的语法糖
* for 是 while 的语法糖

因此添加一个 `desugar` 函数用于脱糖：

```Haskell
desugar :: Statement -> DietStatement
desugar (Incr name)      = (DAssign name (Op (Var name) Plus (Val 1)))
desugar (For s1 e s2 s3) = (DSequence (desugar s1) (DWhile e (DSequence (desugar s3) (desugar s2))))
desugar (Assign name e)  = (DAssign name e)
desugar (If e s1 s2)     = (DIf e (desugar s1) (desugar s2))
desugar (While e s)      = (DWhile e (desugar s))
desugar (Sequence s1 s2) = (DSequence (desugar s1) (desugar s2))
desugar Skip             = DSkip
```

## 语句求值

表达式求值，将得到一个值，而语句求值呢？

语句求值不体现在其结果上，而是体现在它对 **全局状态** 的修改上（仅对 SImPL 而言）。

>表达式 vs 语句的重要区别。

通过对 `Statement` 脱糖，获取更加精简的语言内核 `DietStatement`，我们需要计算 `DietStatement` 组成的语句：

```Haskell
evalSimple :: State -> DietStatement -> State
evalSimple st (DAssign name e)  = extend st name (evalE st e)
evalSimple st (DIf e s1 s2)     = case (evalE st e) of
  1 -> evalSimple st s1
  0 -> evalSimple st s2
evalSimple st (DWhile e s)      = case (evalE st e) of
  1 -> evalSimple (evalSimple st s) (DWhile e s)  // 注意 s 求值将影响 State！
  0 -> st
evalSimple st (DSequence s1 s2) = evalSimple (evalSimple st s1) s2
evalSimple st DSkip             = st
```

`evalSimple` 没有涉及 increment 和 for 两个语句，因此更加简单，这也是语法糖的精髓：方便用户，且不影响语言核心！

最后，用户编写的 SImPL 是由 `Statement` 组成的，因为他们想要使用更甜的 increment 和 for，所以需要实现对 `Statement` 语句的求值：

```Haskell
run :: State -> Statement -> State
run st s = evalSimple st (desugar s)
```

## 总结

[全部代码](https://github.com/satansk/cis-194/blob/master/chapter3/hw3.hs) 只有 100 行左右，就实现了一个五脏俱全的（状态、表达式求值、语法糖、语句求值）解释器，且 SImPL 能实现很多功能，比如求阶乘、平方根等等，并非弱鸡玩具。

更多类似内容可以参考两本书，都非常经典：

* [Implementing functional languages: a tutorial](https://www.microsoft.com/en-us/research/publication/implementing-functional-languages-a-tutorial/?from=http%3A%2F%2Fresearch.microsoft.com%2Fen-us%2Fum%2Fpeople%2Fsimonpj%2Fpapers%2Fpj-lester-book%2F)
* [The Implementation of Functional Programming Languages](https://book.douban.com/subject/1963318/)
* [Essentials of Programming Languages, 3rd Edition](https://book.douban.com/subject/3136252/)

其中前两本以 Haskell 为例，第三本以 Racket 为例，但也很容易用 Haskell 实现。
