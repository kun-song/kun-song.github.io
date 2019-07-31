---
title: HashMap | 哈希碰撞
date: 2018-12-02 23:20:59
tags:
  - 哈希碰撞
categories: Java
---

密码学中有哈希碰撞的说法，而 `HashMap` 中也有哈希碰撞的叫法，两者是否相同？可能不少同学有过类似的疑问，本文将简单解答该问题。

首先看下维基百科对[碰撞](https://en.wikipedia.org/wiki/Collision_(computer_science)的定义：

>In computer science, a collision or clash is a situation that occurs when two distinct pieces of data have the same hash value, checksum, fingerprint, or cryptographic digest.

<!-- more -->

即两个不同值具备相同的哈希值，哈希值通过哈希函数计算而来，维基百科对[哈希函数](https://en.wikipedia.org/wiki/Hash_function)定义如下：

>A hash function is any function that can be used to map data of arbitrary size to data of a fixed size.

哈希函数是将任意大小的数据集 A 映射到固定大小的数据集 B（B 小于 A），因此无论多好的哈希算法都会存在碰撞，这就是密码学中的哈希碰撞。

但 `HashMap` 中的哈希碰撞究竟指什么呢？

我们知道 Java 中每个对象（包括 key）都有 `hashCode` 函数，而 `HashMap` 中也有 `hash` 函数，用于计算 key 的哈希值：

```Java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

有了 key 的哈希值，还需要通过如下方式计算 key 所在桶的索引：

```Java
i = (n - 1) & hash
```

从 key 的 `hashCode` 到 `hash`，再到 `(n - 1) & hash`，key 被映射了 3 次，最终结果是桶索引，因此 `HashMap` 中的哈希冲突是指 **不同 key 被映射到同一个桶**。

而不是不同 key 的 `hashCode` 相同，或不同 key 经过 `hash` 函数的计算结果相同，当然这两者也属于哈希碰撞，而且会影响最终的桶索引计算，但在 `HashMap` 语境中，哈希碰撞不是指它们。

如果每个桶只有一个元素，即元素均匀分布在数组中，此时 `HashMap` 的查找复杂度为 `O(1)`，因此降低哈希碰撞是提升 `HashMap` 效率的关键手段。

Java 8 通过扰动函数降低哈希碰撞概率，在 `hash` 函数中：

```Java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

>`(h = key.hashCode()) ^ (h >>> 16)` 可以保证 32 位的哈希码任意一位发生变化，则 `hash` 结果不同。

至于效果，参考该 [知乎答案](https://www.zhihu.com/question/20733617)，随机选 252 个字符串，且他们的 `hashCode` 没有碰撞，若直接用低位掩码，数组长度为 512 时，没有扰动时，发生 103 次碰撞，使用扰动后，降低为 92 次，扰动肯定是有效果的，但它也不是银弹，改善效果有限。

从该实验也可以看到，`HashMap` 中的哈希碰撞是非常频繁的，密码学中的哈希碰撞概率要小得多。
