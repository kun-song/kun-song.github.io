---
title: Linux | free
date: 2018-12-22 00:41:59
tags:
  - Linux
  - free
categories: Linux
---

`free` 用于查看系统已用内存、空闲内存，man 手册介绍如下：

>free  displays  the  total  amount of **free** and **used physical** and **swap** memory in the system, as well as the **buffers** and **caches** used by the kernel.
>
>The information is gathered by parsing /proc/meminfo.

短短的两句话，信息量却很大：

<!-- more -->

* 信息来源：`/proc/meminfo` 
* 内存类型：
  + physical memory
  + swap memory
  + buffer
  + cache
* 维度：
  + 已用
  + 空闲

默认显示如下：

```
[root@CTU1000073296 beta]# free
              total        used        free      shared  buff/cache   available
Mem:        3344624     1954600      779424      281024      610600      823232
Swap:       4194300     3930932      263368
```

各列含义：

* `total`: Total installed memory (MemTotal and SwapTotal in /proc/meminfo)
* `used`: Used memory (calculated as **total - free - buffers - cache**)
* `free`:  Unused memory (MemFree and SwapFree in /proc/meminfo)
* `shared`: Memory used (mostly) by tmpfs (Shmem in /proc/meminfo, available on kernels 2.6.32, displayed as zero if not available)
* `buffers`: Memory used by kernel buffers (Buffers in /proc/meminfo)
* `cache`: Memory used by the **page cache** and slabs (Cached and Slab in /proc/meminfo)
* `buff/cache`: **Sum** of buffers and cache
* `available`
  + Estimation of how much memory is available for **starting new applications**, without swapping. 
  + Unlike the data provided by the `cache` or `free` fields, this field takes into account page cache and also that not all reclaimable memory slabs will be reclaimed due to items being in use (MemAvailable in /proc/meminfo, available on kernels 3.14, emulated on kernels 2.6.27+, otherwise the same as free)

`free` 默认以 byte 显示，不便于阅读，可通过 `-k/m/g` 指定单位，也可通过 `-h` 将容量自动以合适的单位显示：

```
[root@CTU1000073296 beta]# free -h
              total        used        free      shared  buff/cache   available
Mem:           3.2G        2.1G        537M        296M        550M        547M
Swap:          4.0G        3.6G        426M
```

可以看到 `-h` 后，结果中既有 MB 也有 GB，查看非常方便。

若想查看内存随时间的变化趋势，可通过 `-s` 选项，比如 2s 打印一次： 

```
[root@CTU1000073296 beta]# free -h -s 2
              total        used        free      shared  buff/cache   available
Mem:           3.2G        2.1G        533M        296M        551M        543M
Swap:          4.0G        3.6G        427M

              total        used        free      shared  buff/cache   available
Mem:           3.2G        2.1G        532M        296M        551M        543M
Swap:          4.0G        3.6G        427M

              total        used        free      shared  buff/cache   available
Mem:           3.2G        2.1G        532M        296M        551M        542M
Swap:          4.0G        3.6G        427M
```

前面介绍各字段含义时，有 `buffers` 和 `cache` 字段，但 `free` 默认以 `buf/cache` 显示两者之和，若要分开查看两者，需指定 `-w`（wide 模式）：

```
[root@CTU1000073296 beta]# free -h -w
              total        used        free      shared     buffers       cache   available
Mem:           3.2G        2.2G        471M        302M         15M        543M        483M
Swap:          4.0G        3.6G        428M
```

若要查看各列之和，需指定 `-t`：

```
[root@CTU1000073296 beta]# free -h -t
              total        used        free      shared  buff/cache   available
Mem:           3.2G        2.1G        514M        303M        560M        525M
Swap:          4.0G        3.6G        430M
Total:         7.2G        5.7G        944M
```

