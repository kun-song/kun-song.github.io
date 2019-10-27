---
title: ThreadLocal 之哈希碰撞
date: 2019-10-26 22:23:33
tags:
  - ThreadLocal
categories: Java
---

`ThreadLocalMap` 以 `Entry` 为元素，`ThreadLocal` 实例亲自做哈希表中的 key：

```Java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) { super(k); value = v; }
}
```

任何哈希表都存在哈希碰撞的问题，有两种解决方式：

* 链表法：`HashMap` 等
* 线性探测法：`ThreadLocalMap`

<!-- more -->

## 线性探测法的缺点

`ThreadLocalMap` 采用线性探测法，在内部 k-v 映射被保存在一个数组中，扩缩容时数组大小必须是 2 的倍数：

```Java
private Entry[] table;
```

当待插入位置有元素时，简单粗暴的将 index + 1 寻找下个可用位置，直到找到，优点是简单，缺点是万一发生哈希碰撞，检测效率低：

```Java
private static int nextIndex(int i, int len) { return ((i + 1 < len) ? i + 1 : 0); }

// ThreadLocalMap.set，线性探测可用位置
for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();

    if (k == key) {
        e.value = value;
        return;
    }

    if (k == null) {
        replaceStaleEntry(key, value, i);
        return;
    }
}
```

高效使用线性探测法的前提是 key 的 **哈希值分布地足够均匀**，即 `ThreadLocal` 实例的哈希值均匀的分布在一个 2 的整数倍的数组中，否则通过 index + 1 线性查找的 **效率非常低**。

## threadLocalHashCode

为减少哈希碰撞，`ThreadLocal` 没有使用 `Object.hashCode` 做哈希，而是专门设计了一个哈希字段：

```Java
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

`ThreadLocalMap` 使用 `threadLocalHashCode` 字段作为 `ThreadLocal` 实例的哈希值，假设有 `MyThreadLocal` 子类，其第一个实例的哈希码为 0，后续每个新实例，其哈希码增加 0x61c88647，该数值目的为：

>turns implicit sequential thread-local IDs into **near-optimally spread** multiplicative hash values for power-of-two-sized tables.

即几乎完美地让 `threadLocalHashCode` 均匀分布在 2^x 大小的表中，避免执行线性探测，提升效率。

0x61c88647 计算方式如下，是斐波那契哈希的一个例子：

>0x61c88647 = 1640531527 ≈ 2 ^ 32 * (1 - 1 / φ), φ = (√5 + 1) ÷ 2

## 定址函数

`ThreadLocal` 各种操作本质都要回归到以 `ThreadLocal` 实例在 `ThreadLocalMap` 中进行查找，假设实例为 `x`，查找过程为：

>`x` -> `int hash = x.threadLocalHashCode` -> `f(hash)`

`f` 为根据哈希值计算实例所属位置的定址函数：

```Java
int i = key.threadLocalHashCode & (len-1)
```

其中 `len` 为 `table` 数组长度，必须是 2^x，因此 `len` -1 是一串 1，`threadLocalHashCode & (len-1)` 即截取 `threadLocalHashCode` 的低 N 位数值，由于某些数学原理，这么算出来的位置就是非常均匀，就是能减少哈希碰撞，不服不行。

## 总结

1. `ThreadLocalMap` 采用线性探测法实现哈希表；
2. `ThreadLocal.threadlocalHashCode` 经过专门设计，用于 `ThreadLocalMap` 中以减少哈希碰撞；
3. 定址函数：`int i = key.threadLocalHashCode & (len-1)`；

---

参考：

* [What is the meaning of 0x61C88647 constant in ThreadLocal.java](https://stackoverflow.com/questions/38994306/what-is-the-meaning-of-0x61c88647-constant-in-threadlocal-java)
* [Why 0x61c88647?（好文）](https://www.javaspecialists.eu/archive/Issue164.html)
