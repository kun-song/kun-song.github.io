---
title: Java 语法糖
date: 2019-10-03 14:47:31
tags:
categories: Java
---

Wikipedia 对 [语法糖](https://en.wikipedia.org/wiki/Syntactic_sugar) 定义为：

>In computer science, syntactic sugar is syntax within a programming language that is designed to make things easier to read or to express. It makes the language "sweeter" for human use: things can be expressed more clearly, more concisely, or in an alternative style that some may prefer.
>
>A construct in a language is called "syntactic sugar" if it can be removed from the language **without any effect** on what the language can do: **functionality and expressive power will remain the same**.

即语法糖对语言自身的功能、表达力没有任何影响（增强 or 减弱），它存在的意义只是“更便于”程序员使用，如更容易、清晰、准确地表达逻辑，从而增强代码的可读性，减少出错几率。

Java 的语法糖全部由 Javac 编译器实现，在 `.java` 源文件 -> `.class` 字节码文件的转换过程中，语法糖被“还原”为更基础、原始的语法实现，JIT 编译器执行时，完全 **无法** 感知语法糖的存在。

<!-- more -->

Java 常见的语法糖有：

1. 自动装箱、拆箱
2. try-with-resource
3. 泛型
4. foreach
5. 变长参数
6. 枚举
7. 内部类

了解语法糖最好方式是看它由哪些基础语法实现，目的是解决什么问题，下面一一分析。

## 自动装箱、拆箱

Java 并非一切皆对象，原始类型（如 `int`、`double`）字面值就不是对象，但 Java 容器要求元素必须是对象，如 `List<int>` 非法，必须得 `List<Integer>`，自动装箱、拆箱就是自动进行诸如 `int` 字面值和 `Integer` 对象之间的转换：

```Java
Integer x = 1;
int y = x + 99;
```

使用 `jad -p` 反编译结果为：

```Java
Integer integer = Integer.valueOf(1);
int i = integer.intValue() + 99;
```

即 Javac 根据实际需要 `Integer` 还是 `int`，自动调用两个方法完成两者转换：

* `int` -> `Integer`: `Integer.valueOf` 方法；
* `Integer` -> `int`: `Integer.intValue` 方法；

这也部分解释了为什么 `Integer` 不能用于 `synchronized`：

```Java
Integer lockInt = 100;
```

实际被 desugar 为：

```Java
Integer lockInt = Integer.valueOf(100);
```

而 `Integer.valueOf` 对 [-128, 127] 之间的字面值使用缓存的 `Integer` 对象表示：

```Java
// [-128, 127] 之间的值用缓存
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

即同一 JVM 中，每次 `Integer.valueOf(100)` 返回的都是 **同一对象**，即可能散落各处的 `Integer lockInt = 100` 实际使用的是同一 `Integer` 对象。
