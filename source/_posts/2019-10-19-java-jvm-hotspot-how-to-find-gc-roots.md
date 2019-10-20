---
title: HotSpot | 枚举根节点
date: 2019-10-19 11:02:04
tags:
categories: JVM
---

枚举根节点即枚举 GC root 集合，这是垃圾回收万里长征的第一步。

<!-- more -->

## 速度要快：Stop The World

枚举根节点必须在一个能 **确保一致性的快照** 中执行，分析期间整个系统的引用关系不能发生变化，因此枚举根节点期间，必须 **暂停所有用户线程**，即 Stop The World。

为了减少对用户线程的影响，枚举根节点越快越好。

## 难点：无法扫描所有引用

垃圾回收要准确，要求 GC root 集合必须 **完备**，前面说过可作为 GC root 的引用有几类：

1. 执行上下文（栈）：栈帧中的局部变量表
2. 全局性引用（方法区）：类静态字段 & 常量

其中 **栈** 空间很小，即使全量扫面局部变量表中的引用也能接受，但 **方法区** 动辄几百兆，要逐个扫描其中的引用，耗时太久。

因此每次全量扫描所有引用不可行。

## HostSpot 解决之道：OopMap

既然不能全量扫描系统中的引用（方法区、局部变量表等），那就必须有地方直接记录哪些引用是 GC root。

HotSpot 使用 OopMap 数据结构记录系统 **全部引用及其类型**，枚举 GC root 的时候可以直接使用 OopMap 的内容过滤出 GC root，避免每次都要实时扫描，大大加快枚举速度。

更多关于 OopMap 的内容可参考 [What does Oop Maps means in Hotspot VM exactly](https://stackoverflow.com/questions/26029764/what-does-oop-maps-means-in-hotspot-vm-exactly)，讲的很好。
