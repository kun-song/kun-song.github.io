---
title: 头等函数
tags:
  - Scala
  - fp
categories: 技术
abbrlink: 35818
date: 2018-05-16 20:38:12
---

若在某编程语言中：

* 函数可以作为 **函数参数**
* 函数可以作为 **函数返回值**
* 可以将函数 **赋值** 给变量
* 可以将函数 **存储** 在数据结构（`List` `Map` 等）中
* 支持匿名函数/函数字面值（部分编程语言专家这样认为）

则称该语言具有头等函数（first-class function），具备头等函数的语言中，函数值与 **普通值**（如 `Int` `String` 值）无任何区别，它们仅仅是函数类型（在 Scala 中，函数类型为 `FunctionN`）的值而已。

<!-- more -->

## 头等函数实现难点

对函数式编程语言而言，头等函数是必须品，因为函数式编程风格大量使用高阶函数，若不支持 first-class function，则高阶函数就无从说起了。

但实现头等函数并不容易，历史上把实现头等函数遇到的难题称为 [Funarg problem](https://en.wikipedia.org/wiki/Funarg_problem)（funarg 即 function argument），即：

>**嵌套函数** 或 **匿名函数** 中可以 **直接引用**（即不是通过参数传递方式引用）该函数 **定义时** 上下文环境中的标识符，但该标识符可能不在该函数 **调用时** 的上下文环境中，如何处理该场景？

通俗地讲，即函数 `f` 在函数 `g` 的函数体内定义，且 `f` 引用了自己局部变量以外的自由变量，之后 `f` 作为 `g` 的返回值返回，然后调用 `f`，此时 `f` 已经脱离了定义它的上下文环境，如何处理它引用的自由变量呢？

早期的命令式编程语言通过袋鼠策略绕过了该问题：

* 不支持函数作为返回值（Algol），或
* 禁止嵌套函数，进而消除非局部变量引用（C 语言）

早期的函数式编程语言 Lisp 通过 [动态作用域](https://en.wikipedia.org/wiki/Dynamic_scoping) 解决该问题：

* 在函数 **调用时**（而非定义时）的上下文环境中查找自由变量的值

动态作用域是设计败笔，因此 Scheme 最早引入 [词法作用域](https://en.wikipedia.org/wiki/Lexically_scoped) 解决该问题：

* 将函数引用视为 **闭包**，而非简单的 **函数指针**，闭包中保存两部分内容：
  + 函数 **定义时** 的自由变量
  + 函数代码

## 不同语言对头等函数的支持程度

头等函数并非函数式编程语言专属，各编程语言家族或多或少都支持一些头等函数特性，例如：

* 函数式语言
  + Scheme ML Haskell F# Scala 等支持 first-class function 的所有特性
  + Lisp 由于动态作用域的错误设计，并没有正确理解 first-class function 的正确行为
* 脚本语言
  + 如 Perl Python Lua JavaScript
* 命令式语言（Funarg 问题由返回嵌套定义的函数引起，命令式语言通过不同方式绕过该问题）：
  + C 语言家族支持函数作为参数、返回值，但禁止嵌套函数，以避免 Funarg problem
  + Algol 支持函数作为参数、嵌套函数，但不支持函数作为返回值，同样是为了避免 Funarg problem

通常认为头等函数的价值在于：

* 返回值返回嵌套定义的、捕获了非局部变量的函数

对命令式语言而言，它们或不支持函数返回值（Algol），或不支持嵌套函数（C 语言），因此一般不认为其具有头等函数。

---

参考：

* 维基百科 [First-class function](https://en.wikipedia.org/wiki/First-class_function)
* 维基百科 [Gunarg problem](https://en.wikipedia.org/wiki/Funarg_problem)