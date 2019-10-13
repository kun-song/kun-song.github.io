---
title: 特殊方法
date: 2019-10-13 20:31:32
tags:
categories: JVM
---

>[Special Methods](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.9)

在 Java 语言维度，有两种初始化:

1. 实例初始化
2. 类（含接口）初始化

有很多语言级特性可以完成这两种初始化：

* 构造方法
* 普通字段初始化器 & `static` 字段初始化器
* `{}` 块 & `static {}` 块

对应到 JVM 维度，同样只有实例、类（含接口）两种初始化，不过它们各自只有一个方法：

* `<init>`：实例初始化
* `<clinit>`：类初始化

<!-- more -->

这两个方法由 JVM 提供，且由于它们并非合法的 Java 标识符，所以在 Java 语言里无法直接使用，只能由 JVM 通过 `invokespecial` 指令使用。

Java 语言维度的初始化代码会被 JVM 统一归约到 `<init>` 或 `<clinit>` 方法中，各个代码块有不同的 **优先级**，即在 `<init>` 或 `<clinit>` 中的 **位置顺序**，JVM 执行实例 or 类的初始化时，就是简单粗暴的执行 `<init>` 或 `<clinit>`，它根本无法感知语言维度的构造器、代码块等概念的存在。
