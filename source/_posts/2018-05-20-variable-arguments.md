---
title: 变长参数
tags:
  - Scala
  - Java
categories: 技术
abbrlink: 15952
date: 2018-05-20 22:27:44
---

变长参数是很常见的语言特性，Java 和 Scala 都支持，使用很简单，本文介绍其在 Java 和 Scala 中的基本用法。

<!-- more -->

## Java 变长参数

Java 5 引入对变长参数的支持，并规定变长参数必须满足：

1. 最多只能有一个变长参数
2. 必须是参数列表中的 **最后一个** 参数

变长参数只是数组参数的语法糖，更加方便而已：

```Java
void foo(String... args);
void foo(String[] args);
```

* 两个 `foo` 定义完全相同，不能作为重载函数同时存在！

例子：

```Java
private static int sum(int ... xs) {
  int result = 0;
  for (int i : xs) {
    result += i;
  }

  return result;
}
```

可以使用数组调用 `sum`：

```Java
int[] xs = { 1, 2, 3 };
sum(xs);
```

## Scala 变长参数

Scala 同样支持变长参数，也必须满足同样的两个条件：

1. 最多一个
2. 只能是参数列表中最后一个

例如：

```Scala
def sum(xs: Int*): Int = xs.sum
```

* Scala 将 `xs` 作为 `Array[Int]` 处理

与 Java 类似，Scala 将变长参数作为序列（可变 `Array` ）来处理；与 Java 不同的是，虽然 `sum` 内部将 `xs` 作为序列处理，但却无法直接使用序列调用 `sum` 函数：

```Scala
sum(List(1, 2, 3))  // 编译报错
```

使用 `_:*` 标识可以使用序列（`List` `Vector` `Seq` `Array`）来调用 `sum` 函数：

```Scala
sum(List(1, 2, 3): _*)
sum(Vector(1, 2, 3): _*)
sum(Array(1, 2, 3): _*)
sum(Seq(1, 2, 3): _*)
```

* `_:*` 告诉编译器，将序列分解为 **离散元素**

很多集合类的构造函数都使用了变长参数，例如 `List` `Map` 等。

## Java、Scala 互操作

默认情况下，Java 无法使用 Scala 定义的具备变长参数的方法，除非为其添加 `scala.annotation.varargs` 注解。

`@varargs` 注解使 Scala 编译器生成 **两个方法**：

* 一个给 Scala 使用（字节码参数是一个 Seq）
* 一个给 Java 使用（字节码参数是一个数组）

例如：

```Scala
@scala.annotation.varargs
def select(exprs: Expression*): DataFrame = { ... }
```
