---
title: The Towers of Hanoi -- 递归的力量
date: 2018-06-21 23:34:05
tags:
  - Haskell
  - 递归
  - 算法
  - CIS 194
categories: 公开课
---

汉诺塔问题是一个非常经典的可以用递归优美解决的问题。

问题目标：将盘子从第一个柱子移动到最后一个柱子上：

<img src="/images/hanoi-objective.png" alt="汉诺塔问题的目标" style="width: 400px;"/>

<!-- more -->

不允许把大盘子放在小盘子上面：

<img src="/images/hanoi-rule.png" alt="汉诺塔问题的目标" style="width: 400px;"/>

直接贴下算法：

把 `a` 上的 `n` 个盘子移动到 `c` 上，需以 `b` 做“缓冲”：

1. 以 `c` 做缓冲，将 `a` 上的 `n-1` 个盘子移动到 `b` 上
2. 将 `a` 最底下的盘子移动到 `c` 上
3. 以 `a` 做缓冲，将 `b` 上的 `n-1` 个盘子移动到 `c` 上

用 Haskell 实现该算法：

```Hakell
type Peg = String
type Move = (Peg, Peg)

hanoi :: Integer -> Peg -> Peg -> Peg -> [Move]
hanoi 0 _ _ _ = []
hanoi n a b c = (hanoi (n - 1) a c b) ++ ((a, c) : hanoi (n - 1) b a c)
```

算法实现只有两行，两行代码解决了人脑无法处理的问题，现在想来依旧很震撼。

---

参考：

* [CIS 194 week 1 homework](http://www.seas.upenn.edu/~cis194/spring15/hw/01-intro.pdf)
