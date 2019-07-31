---
title: 译 | Guide to the Most Important JVM Parameters
date: 2018-12-23 15:21:19
tags:
categories: JVM
---

原文在 [这里](https://www.baeldung.com/jvm-parameters)。

本文介绍配置 JVM 的几个关键选项。

## 堆内存

```
-Xms<heap size>[unit]
-Xmx<heap size>[unit]
```

其中 `unit` 可以是 `g/m/k` 等，例如：

<!-- more -->

```
-Xms2g -Xmx5g
```

按 [Oracle 官方指南](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html)，除堆的总容量（Xms/Xmx）外，第二重要的设置是新生代的内存占比，默认情况下，新生代最小 1310M，无上限，也可以显式设置：

```
-XX:NewSize=<young size>[unit]
-XX:MaxNewSize=<young size>[unit]
```

## 垃圾收集器

选择合适的垃圾收集算法也很重要，JVM 有 4 种 GC 实现：

* Serial Garbage Collector
* Parallel Garbage Collecor
* GMS Garbage Collecor
* G1 Garbage Collecor

分别通过如下选项指定：

```
-XX:+UseSerialGC
-XX:+UseParallelGC
-XX:+UseParNewGC
-XX:+UseG1GC
```

## GC 日志

```
-XX:+UseGCLogFileRotation 
-XX:NumberOfGCLogFiles=< number of log files > 
-XX:GCLogFileSize=< file size >[ unit ]
-Xloggc:/path/to/gc.log

-XX:+PrintGCTimeStamps 
-XX:+PrintGCDateStamps
```

## 处理内存不足

线上应用若出现 `OutOfMemoryError`，可能直接崩溃，这类错误很难复现，因此出现该错误时转储堆信息（dump heap memory）非常重要：

```
-XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=./java_pid<pid>.hprof
-XX:OnOutOfMemoryError="< cmd args >;< cmd args >" 
-XX:+UseGCOverheadLimit
```

`HeapDumpOnOutOfMemoryError` 开启转储功能，`OnOutOfMemoryError` 类似回调，当 `OutOfMemoryError` 发生时执行指定指令，例如重启：

```
-XX:OnOutOfMemoryError="shutdown -r"
```

## 32/64 位

若 OS 上同时存在 32 位和 64 位的 JDK，则 JVM 默认选择 32 位，可以手动指定为 64 位：

```
-d64
```

## 杂项

* `-server`：使用 Server Hotspot VM，注意 64 位 JVM 默认开启 `-server`；
* `-XX:+UseStringDeduplication`：Java 8u20 引入，将相同 `String` 实例转化为全局 `char[]`，减少重复内存消耗；
* `-XX:+UseLWPSynchronization`：使用 LWP(light weight process)-based 同步策略，而非 thread-based 同步；
* `-XX:LargePageSizeInBytes`：sets the large page size used for the Java heap; it takes the argument in GB/MB/KB; with larger page sizes we can make better use of virtual memory hardware resources; however, this may cause larger space sizes for the PermGen, which in turn can force to reduce the size of Java heap space.
* `-XX:MaxHeapFreeRatio`：GC 后堆空闲内存的最大比例，防止 shrink；
* `-XX:MinHeapFreeRatio`：GC 后堆空闲内存的最小比例，防止 expansion；
* `-XX:SurvivorRatio`：eden/survivor 比例，例如 `-XX:SurvivorRatio=5` 则新生代分为 7 分，eden 占 5，两个 survivor 占 2；
* `-XX:+UseLargePages`：use large page memory if it is supported by the system; please note that OpenJDK 7 tends to crash if using this JVM parameter
* `-XX:+UseStringCache`：缓存常用的字符串；
* `-XX:+UseCompressedStrings`：用 `bye[]` 而非 `char[]` 表示字符串；
* `-XX:+OptimizeStringConcat`：优化字符串拼接操作；

