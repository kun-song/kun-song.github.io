---
title: JVM | GC 日志
date: 2018-12-20 08:42:42
tags:
  - Java
  - JVM
  - jstat
  - GC log
categories: JVM
---

看懂 GC 日志是 Java 司机的必备技能，本文简单总结 GC 日志的相关内容。

首先，GC 日志有两种获取方式：

* 通过 jstat 动态查看
* 通过 JVM 参数打印

<!-- more -->

## 获取方式

### 动态打印

jstat（JVM Statistics Monitoring Tool）可监控（本地/远程）JVM 运行时状态，包括类加载、内存、垃圾收集、JIT 编译等运行数据，仅提供命令行界面，是运行时定位 JVM 性能问题的首选工具。

jstat 有很多 GC 相关的参数，例如 `-gc` 将监控堆（Eden 区、两个 survivor 区、老年代、永久代等）的容量、已用容量、GC 时间等信息：

```
> $ jstat -gc 25421
  S0C     S1C     S0U    S1U      EC      EU         OC         OU       MC       MU      CCSC    CCSU      YGC    YGCT   FGC     FGCT     GCT
31232.0 33792.0 24717.6  0.0   456704.0 89791.4   883712.0   113490.0  157952.0 110577.1 20992.0 15124.6    274   14.908  19     18.890   33.798
```

各个字段的含义在 jstat 的 man 手册中解释的很清楚：

```
+-------+-------------------------------------------+
|Column |                Description                |
+-------+-------------------------------------------+
|SOC    | Current survivor space 0 capacity (KB).   |
|S1C    | Current survivor space 1 capacity (KB).   |
|S0U    | Survivor space 0 utilization (KB).        |
|S1U    | Survivor space 1 utilization (KB).        |
|EC     | Current eden space capacity (KB).         |
|EU     | Eden space utilization (KB).              |
|OC     | Current old space capacity (KB).          |
|OU     | Old space utilization (KB).               |
|PC     | Current permanent space capacity (KB).    |
|PU     | Permanent space utilization (KB).         |
|YGC    | Number of young generation GC Events.     |
|YGCT   | Young generation garbage collection time. |
|FGC    | Number of full GC events.                 |
|FGCT   | Full garbage collection time.             |
|GCT    | Total garbage collection time.            |
+-------+-------------------------------------------+
```

### JVM 参数

GC 日志相关的 JVM 参数：

* `-XX:+PrintGC` 输出 GC 日志
* `-XX:+PrintGCDetails` 输出 GC 详细日志
* `-XX:+PrintGCTimeStamps` 输出 GC 的时间戳（以基准时间形式）
* `-XX:+PrintGCDateStamps` 输出 GC 的时间戳（以日期形式，如 2013-05-04T21:53:59.234+0800）
* `-XX:+PrintHeapAtGC` 在 GC 前后打印堆信息
* `-Xloggc:../logs/gc.log` GC 日志的输出路径

## 字段解释

每种垃圾收集器打印的 GC 日志各有不同，但 JVM 设计者尽量保持了一些共性，方便用户阅读，例如：

```
0.756: [Full GC (System) 0.756: [CMS: 0K->1696K(204800K), 0.0347096 secs] 11488K->1696K(252608K), [CMS Perm : 10328K->10320K(131072K)], 0.0347949 secs] [Times: user=0.06 sys=0.00, real=0.05 secs]
1.728: [GC [ParNew: 38272K->2323K(47808K), 0.0092276 secs] 39968K->4019K(252608K), 0.0093169 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
GC 时间       区域     GC 前后该区域内存变化      本次 GC 耗时     GC 前后堆的内存变化          总耗时
```

* 0.756、1.728：GC 发生的时间，以 JVM 启动经过的秒数计算；
* `[Full GC`、`[GC`：本次 GC 的停顿类型，（并非指 GC 发生在新生代还是老年代）：
  + `Full GC` 表示发生 Stop The World
  + `Full GC(system)` 表示通过 `System.gc()` 触发 GC
* `[DefNew`、`[Tenured`、`[Perm`、`[CMS`、`[ParNew` 表示 GC 发生的区域，区域名字与 GC 收集器密切相关，例如：
  + 新生代：
    - Serial 收集器称之为 DefNew
	- ParNew 收集器称之为 ParNew
	- Parallel Scavenge 收集器称之为 PSYoungGen
* `[]` 内 `38272K->2323K(47808K)`：GC 前 **该区域** 已用容量 -> GC 后该区域已用容量（该区域总容量）
* `[]` 外 `39968K->4019K(252608K)`：GC 前 **堆** 已用容量 -> GC 后堆已用容量（堆总容量）
* `[Times: user=0.01 sys=0.00, real=0.00 secs]` GC 耗时分布：
  + CPU 耗时：
    - `user` 用户态耗时
	- `sys` 内核态耗时
  + 墙钟时间（wall clock time）耗时
  
注意，与 CPU 时间不同，墙钟时间包含等待时间，如等待磁盘 IO、线程阻塞等等，因此单核机器的墙钟耗时一般大于 CPU 耗时，但在多核机器上，各个 CPU 耗时相加后，可能超过墙钟耗时。


