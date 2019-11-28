---
title: 类加载（八）：forName 与 loadClass
date: 2019-11-25 23:55:17
tags:
  - 类加载
categories: JVM
---

触发类加载的方式分为两类：

* 隐式加载：通过 **主动引用** 触发，见《类加载（二）：何时加载类？》
* 显式加载
  * `Class.forName`
  * `ClassLoader.loadClass`

<!-- more -->

类加载机制分为 3 个大阶段：

1. 加载
2. 连接（验证、准备、解析）
3. 初始化

`forName` 和 `loadClass` 两者都接受一个 `String` 类型的全限定名，并返回代表该类的 `Class` 实例，两者的区别为：

* `forName`：加载 + 连接 + （可选的）初始化
* `loadClass`：加载

`forName` 可完成完整的类加载过程，包括执行类初始化代码 `<clinit>`，而 `findClass` 仅根据 **全限定名** 加载类的二进制字节流，并生成 `Class` 实例。

基本可依据是否需执行 `<clinit>` 方法，即 **静态字段初始化 + `static` 块** 来进行选择。
