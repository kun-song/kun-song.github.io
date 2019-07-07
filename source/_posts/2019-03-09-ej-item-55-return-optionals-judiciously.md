---
title: Item 55 | Return optionals judiciously
date: 2019-03-09 14:39:58
tags:
  - Java
categories: Java
---

方法并非在所有场景下都能返回值，例如 `max` 返回集合中的最大元素，若集合为空，则不存在合理的返回值。

Java 8 之前，无法返回合理值时，有两种处理手段：

1. 抛出异常
2. 返回 `null`

这两种方式都不够好，首先异常应该用于真正的异常场景（空集合并非异常场景），而且异常的 **开销** 较大，因为创建异常时需要捕获整个调用栈；返回 `null` 避免了异常的这些问题，但它也有自己的缺点：调用者必须特殊处理返回 `null` 的场景，若忘记处理，则 `null` 值可能被保存在某数据结构中，最后在看似毫不相干的地方抛出空指针异常，排查困难。

<!-- more -->

Java 8 给出了第 3 种手段：

* 返回 `Optional`

`Optional` 作用类似 checked exception，都 **强制** 用户处理特殊情况，而 unchecked exception 和 `null` 不强制用户处理，用户可选择忽略处理。

`Optional` 使用有以下几点值得注意：

* `Optional` 存在一定性能开销，若性能要求特别高，应使用异常或返回 `null`；
* 相对 `orElse`，`orElseGet` 用于默认值创建非常耗时的场景；
* `Optional` 不能包裹容器类型，例如集合、列表、`map`、`stream`、数组、`Optional` 等，因为空集合本身即可用作无合理值的返回值，增加 `Optional` 一层后，反而增加处理复杂度，比如 `Optional<emptyList>` 和 `emptyList` 有啥区别，需要特殊处理吗？
* `Optional` 不能包裹 `Integer`、`Long`、`Double` 等包装类型，因为存在性能惩罚，应该用 `OptionalInt`、`OptionalLong` 等替代；
* `Optional` 可以包裹 `Boolean`、`Byte`、`Character`、`Short` 和 `Float`，因为它们占用空间较少；
* `Optional` 最合理的使用场景为用作返回类型，其他用法基本不合理，如：
  + 用作 `map` 中的 key 或 value
  * 用作集合元素
  * 类型的字段（部分合理，需要特殊审视）
