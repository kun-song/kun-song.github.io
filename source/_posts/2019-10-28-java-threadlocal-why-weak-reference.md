---
title: ThreadLocal 之弱引用
date: 2019-10-28 14:50:11
tags:
  - ThreadLocal
categories: Java
---

`ThreadLocal` 本身 **并非存储数据的容器**，**真正的数据** 保存在每个线程的 `Thread.threadLocals` 哈希表中，`ThreadLocal` 作用有两点：

* `ThreadLocal` 类定义了操作 `threadLocals` 的各种操作，是存取 `threadLocals` 的入口；
* `ThreadLocal` 实例作为 `threadLocals` 中映射的 **键**；

本文主要关注第二点 `ThreadLocal` 实例对象用作 key 的细节。

<!-- more -->

## Entry：弱引用

`Thread.threadLocals` 哈希表类型为 `ThreadLocal.ThreadLocalMap`，是一个专门为该场景设计的哈希表，它的元素为：

```Java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) { super(k); value = v; }
}
```

对比 `HashMap` 的元素类型：

```Java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
```

可以发现 `Entry` 是一个指向 `ThreadLocal` 实例的弱引用，不过该弱引用中定义了一个 `value` 字段，因此可以认为 `Entry` 类型为：

>`ThreadLocal` 实例 -> value 的映射

## 弱引用的“不良”后果

仅被弱引用指向的实例生命周期很短，每次 GC 时必然会被回收，因此假设方法 `f` 中有：

```Java
ThreadLocal<String> msg = ThreadLocal.withInitial(() -> "Hello");
```

线程 `t` 执行完该语句后，该线程内部的 `t.threadLocals` 哈希表中就会增加一个类型为 **弱引用** 的 `Entry` 实例：

* 该弱引用指向 `msg` 实例；
* 该弱引用内容为 `msg` 实例 -> `"Hello"`；

到目前为止，一共有 2 个引用指向 `ThreadLocal` 实例：

1. 强引用 `msg`；
2. 弱引用 `Entry`，假设名字为 `e`；

线程 `t` 执行完方法 `f` 后，由于 `msg` 是局部变量，在栈帧出栈后即被垃圾回收，变成 `null`，此时只有 1 个引用指向该实例：

* 弱引用 `e`；

只被弱引用指向的实例，将在下次 GC 时被回收，因此该 `ThreadLocal` 实例在下次 GC 后被回收，此后 `t.threadLocals` 的元素 `e` 内容变为：

>`null` -> `"Hello"`

由于 `e` 本身被 `t.threadLocals` 强引用，所以 `e` 会一直存在于该线程的 `threadLocals` 哈希表中，但它指向的 `ThreadLocal` 实例被回收了。

到目前为止，已经无法通过执行 `msg.remove()` 删除 `e` 了，因为 `msg` 为 `null`，只有两种情况 `e` 会被删除：

1. 线程 t 通过操作其他 `ThreadLocal` 实例，比如 `msg2.get/set` 等，`ThreadLocalMap` 内部会自动清除 `null`-`"Hello"` 映射；
2. 线程 t 终止，进而 `t.threadLocals` 被回收变为 `null`，它的所有元素不可达，自然 `null`-`"Hello"` 也会被回收；

如果这两种情况都没发生，则发生 **内存泄漏**，`"Hello"` 永远无法被删除，当然这里用 `"Hello"` 举例不大恰当，如果把 `"Hello"` 替换为超大数组 or 超长字符串，会更清楚。

## 为何不用强引用？

### 强引用更糟糕

既然把 `Entry` 定义为弱引用可能导致内存泄漏，那为何不直接使用强引用？

假设 `Entry` 被定义为强引用，接着上面的分析，线程 t 执行完方法 `f` 后，同样只有一个引用指向 `ThreadLocal` 实例：

* 强引用 `Entry`，假设名字为 `e`；

由于被强引用指向的实例永远不会被垃圾回收，因此 `e` 指向的 `ThreadLocal` 实例 **不会被 GC 回收**，`e` 的内容也不会变为 `null`-`"Hello"`，而是保持为：

>`ThreadLocal` 实例 -> `"Hello"`

由于外部没有指向该 `ThreadLocal` 的实例，同样无法通过 `remove` 删除该映射，`Entry` 为弱引用时的两种删除场景也只剩一种：

1. 线程 t 终止；

执行 `msg2.get/set` 进行间接删除的方式失效，因为已经无法通过 key 为 `null` 来判断 k-v 映射已经可以删除。

因此 `e` 中的 k-v **同样会发生内存泄漏**，并且比 `Entry` 为弱引用时更糟糕：

* `Entry` 为弱引用
  * 作为 key 的 `ThreadLocal` 实例会被 GC 回收，只有 value 会内存泄漏；
  * 有 2 种场景可以删除过期的 k-v；
* `Entry` 为强引用
  * key 和 value 都会内存泄漏；
  * 只有 1 种场景可以删除过期的 k-v，即线程终止；

### 弱引用的优势

`ThreadLocalMap` 的注释有解释：

>To help deal with **very large** and **long-lived** usages, the hash table entries use WeakReferences for keys. However, since reference queues are not used, stale entries are guaranteed to be removed only when the table starts running out of space.

通过将 `Entry` 定义为弱引用，带来两个优势：

1. GC 可以自动回收作为 key 的 `ThreadLocal` 实例，完成 k-v 到 `null`-v 的转换；
2. `null`-v 中的 `null` 可作为过期 k-v 的标志，`ThreadLocalMap` 很多方法会自动删除它们；

因此使用弱引用是设计者仔细考虑过的。
