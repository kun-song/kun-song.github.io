---
title: Racket | 拨开延迟计算的迷雾
date: 2018-05-22 23:18:10
tags:
  - Racket
categories: 技术
---

{% post_link 2018-05-17-call-by-value-call-by-name 传值调用 vs 传名调用 %} 这篇文章简单介绍了 Scala 的传名调用，并提到可以通过无参函数手动实现传名调用。

实际上 Scala 的传名调用符号 `=>` 仅仅是语法糖，为探寻事实的真相，本文用另一种函数式编程语言 Racket 手动实现延迟计算（传名调用仅仅是延迟计算的一种），以破除大家心中的迷雾。

<!-- more -->

## 使用 thunk 实现延迟计算

Racket 采用严格求值策略，即给定函数调用：

```Racket
(e1 e2 e3 ... en)
```

实际调用函数 `e1` 之前，首先对参数 `e2` 到 `en` 求值。

但 Racket 的 `if` 并非严格求值：

```Racket
(if e1 e2 e3)
```

* 根据 `e1` 的计算结果，`e2` 和 `e3` 只有一个被求值

我们尝试实现自己的 `if`：

```Racket
(define (bad-if x y z)
  (if x y z))
```

与真正的 `if` 相比，`bad-if` 会 **同时** 对 `x` `y` `z` 求值，可能导致死循环：

```Racket
(define (bad-fact x y)
  (bad-if (= y 0)
          1
          (* y (bad-fact x (- y 1)))))
```

而我们知道，定义函数时，函数体并不会立刻被计算，只有当 **调用函数** 时才会真正计算函数体。利用该特性，我们可以把 `y` 和 `z` 视为函数，并通过 `(x)` 和 `(z)` 分别调用两者，进而计算它们的函数体，获取真正的数据：

```Racket
(define (good-if x y z)
  (if x (y) (z)))
```

* `(x)` 和 `(y)` 为无参函数的调用

使用 `good-if` 时，第二、三个参数必须是函数形式：

```Racket
(define (good-fact x y)
  (good-if (= y 0)
           (lambda () 1)  ; 函数形式
           (lambda () (* x (good-fact x (- y 1))))))  ; 函数形式
```

>使用 **无参函数** 实现 **延迟计算** 是一种常见惯用法，并称该函数为 thunk。

## 缓存计算结果

使用 thunk 有明显缺点：若函数的计算量巨大，对其 **重复** 求值的开销很大。

Scala 有 `lazy` 关键字用于缓存计算结果，在 Racket 中我们可以使用可变列表单元（mcon cells）达到类似效果：

```Racket
(define (my-delay th) ; 调用时,f 应为 thunk
  (mcons #f th))

(define (my-force p) ; promise
  (if (mcar p) ; 已经计算过了吗?
      (mcdr p) ; 计算过直接用第一次结果就行
      (begin (set-mcar! p #t)
             (set-mcdr! p ((mcdr p))) ; 第一次计算
             (mcdr p))))
```

`my-delay` 接受一个 thunk，并将其封装在 mutable pair（内容为 (标志, thunk)）中。

`my-delay` 则接受 mutable pair，若 `(mcar p)` 为 `false`，即为首次使用，此时：

1. 获取 `(mcdr p)` 存储的 thunk
2. 计算 thunk `((mcdr p))`
3. 将 thunk 结果存入 mutable pair，**覆盖** 原本存在的 thunk，此后只有结果，再无 thunk

若 `(mcar p)` 为 `true` 表明并非第一次使用，直接返回 mutable pair 中缓存的 thunk 计算结果。

例如有一个非常弱智、仅仅使用 thunk 的乘法函数：

```Racket
(define (mult x y-thunk)
  (cond [(= x 0) 0]
        [(= x 1) (y-thunk)]
        [#t (+ (y-thunk) (mult (- x 1) y-thunk))]))
```

对于函数调用 `(mult e1 (lambda () e2))`，若 `e1` 是一个非常大的数字，而 `e2` 每次递归时都会被求值，因此 `e2` 将被求值非常多次，严重耗费性能。

可以使用 `my-delay` 和 `my-force` 重写 `mult` 函数：

```Racket
(define (good-mult x y-delay)
  (cond [(= x 0) 0]
        [#t (+ (my-force y-delay) (good-mult (- x 1) y-delay))]))
```

使用时，将原本第二个参数中的 `y-thunk` 用 `my-delay` 封装起来：

```Racket
(good-mult 100 (my-delay (lambda () (slow-add 3 4))))
```

如此一来，无论 `(good-mult x y)` 中的 `x` 多大，`y` 都只会求值一次。
