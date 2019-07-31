---
title: 闭包初探
tags:
  - 闭包
categories: 技术
abbrlink: 60696
date: 2018-05-11 08:20:01
---

## 闭包定义

### 维基百科

首先建议阅读维基百科对闭包的定义，原文比较长，我试着逐句理解下：

>In programming languages, a closure (also lexical closure or function closure) is a technique for implementing **lexically scoped name binding** in a language with first-class functions.

从概念角度看，在具备 first-class functions 的编程语言中，闭包是一种实现 **词法作用域级别的名字绑定** 的技术。

<!-- more -->

>Operationally, a closure is a record storing a function together with an **environment**.

从实现角度看，闭包是保存 **函数** + **上下文环境** 的记录。

到此为止，从概念、实现上解释了闭包是什么，剩下的是对上下文环境的解释：

>The environment is a mapping associating each **free variable** of the function (variables that are used locally, but defined in an enclosing scope) with the **value or reference** to which the name was bound when the closure was **created**.

（上下文）环境是 **自动变量名字** 与该名字的 **值/引用** 的映射（name -> value），注意变量的值、引用是闭包 **创建** 时确定的。

自由变量：

* 在函数的 enclosing scope 中定义，但在函数内部使用的变量；
* 既不在函数参数列表中，也不是函数局部变量的变量；

>A closure—unlike a plain function—allows the function to access those **captured variables** through the closure's **copies** of their values or references, even when the function is invoked outside their scope.

被捕获变量：即闭包使用的自由变量，闭包中一旦引用自由变量，则称该变量被闭包捕获。

若语言不支持 first-class functions，则闭包与普通函数并无区别，若支持 first-class function 则两者区别巨大：

* 闭包的上下文环境会保存被捕获的自由变量值、引用的 **副本**；
* 闭包作为函数返回值返回后，调用闭包，闭包中依然保存其 **定义时** 的自由变量的值；

#### 总结如下：

* 闭包 = 函数 + 上下文环境
* 上下文环境 = 自由变量名字 -> 值/引用（key -> value）
* 自由变量 = 在函数的 enclosing scope 定义，在函数内部引用的变量
* 上下文环境中保存的 value/reference 是闭包 **定义** 时的 **副本**
* 不同场景中，自由变量的值可以不同，进而闭包也可以不同

### 我的理解

闭包英文是 closure，更精确一点，闭包是 "first-class function with lexical scope"，即具备词法作用域的第一等函数，要理解这句话，首先要明白两个概念：

* 词法作用域
* 第一等函数

稍微有点 FP 基础的同学肯定了解第一等函数，即在编程语言中，函数是第一等公民，与普通值没有任何区别，因此：

* 函数可以用作 **函数参数**
* 也可以用作 **函数返回值**

而词法作用域（lexical scope）可以参考这篇博文 []()。

## 闭包示例

不同语言中的闭包略有不同，虽然按照维基百科对闭包的定义，需要保存自由变量的 **副本**，实际上各个语言的闭包实现各不相同，本文分别演示 Scala、Standard ML 和 Racket 中的闭包。

### Standard ML

```ML

```

### Scala

```Scala
var a = 10
def f(x: Int): Int = a + x

f(10) // 20
```

```Scala
var a = 10
def f(x: Int): Int = a + x

a = 1
f(10) // 11
```

* Scala 闭包不会复制自由变量的值，而是直接绑定到自由变量，因此闭包中对自由变量的修改，将反应在所有引用该变量的地方；

### Racket

```Racket
(define a 10)
(define (f x) (+ a x))

(f 10) ; 20
```

`f` 是闭包，其函数体内引用了自由变量 `a`，当计算 `(f 10)` 时，会计算函数体 `(+ a 10)`，解释器在 `f` 的 environment 中查找 `a` 的绑定，找到了 10，因此最后结果为 20。

Racket 并非纯函数式编程语言，可以用 `set!` 修改绑定：

```Racket
(define a 10)
(define (f x) (+ a x))

(set! a 1)
(f 10) ; 11
```

在计算 `(f 10)` 之前，`a` 被重新绑定到 1，因此计算函数体 `(+ a 10)` 时查找到的 `a` 为 1，最后结果为 11。

## 闭包应用

### 柯里化

柯里化并非什么高深的语言特性，简单讲，柯里化不过是闭包的一种 **惯用法** 而已，从来就没有过魔法。

### 部分应用函数

