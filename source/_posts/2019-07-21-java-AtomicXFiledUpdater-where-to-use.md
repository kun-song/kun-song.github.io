---
title: AtomicXFieldUpdater 使用总结
date: 2019-07-21 12:18:54
tags:
  - AtomicIntegerFieldUpdater
  - AtomicLongFieldUpdater
  - AtomicReferenceFieldUpdater
categories: Java
---

JUC 有 3 种原子更新器：

1. `AtomicIntegerFieldUpdater`
2. `AtomicLongFieldUpdater`
3. `AtomicReferenceFieldUpdater`

它们的用途、实现非常相似，以 `AtomicIntegerFieldUpdater` 为例，它的用途是：

>A reflection-based utility that enables **atomic updates** to designated 
**volatile int** fields of designated classes.

即通过 **反射** 实现对 `volatile int` 类型的字段的 **原子更新**。

<!-- more -->

仔细看过源码可以发现，`AtomicInteger` 和 `AtomicIntegerFieldUpdater` 的公共 API 几乎一模一样，所以在提供 CAS 操作这个层面，两者是几乎等价的。

问题来了，由于反射的性能较低，为什么不直接用 `AtomicInteger`，而要用 `volatile` + `AtomicIntegerFieldUpdater` 呢？

注释中也有解释：

>This class is designed for use in atomic data structures in which **several fields** of the same node are **independently** subject to atomic updates.

但实际上，即使是独立原子更新的多个字段，也可以直接用 `AtomicInteger`，完全没问题！

当然 `AtomicXFieldUpdater` 并非毫无用处，相对原子类而言，有如下优点：

1. `AtomicX` 毕竟是复杂类型，空间占用比 `volatile` 的原始类型要大，在超大数量的场景下，`AtomicXFieldUpdater` 在内存占用方面有优势；
2. 如果某个字段的 **所有操作** 都是原子操作，那可以用 `AtomicX`，但有的场景下，字段既需要原子操作，也需要普通操作，这时可以考虑用 `AtomicXFieldUpdater`；
3. 如果要原子更新的字段在第三方类中，无法直接修改源码，则使用 `AtomicXFieldUpdater`；

## 原子更新器的优势

### 内存消耗小

假设有一个类：

```Java
public class Counter {
    private final AtomicInteger ct = new AtomicInteger(0);
    public int addOne() {
        return ct.incrementAndGet();
    }
}
```

如果系统中只有少量 `Counter` 实例，那没有问题；但如果实例数量有成千上万个，则 `Counter.ct` 字段会占用过多堆内存，造成不必要的内存浪费。

解决方式是用原子更新器代替原子类：

```Java
public class Counter {
    private volatile int ct = 0;

    private static final AtomicIntegerFieldUpdater<Counter> updater =
            AtomicIntegerFieldUpdater.newUpdater(Counter.class, "ct");

    public int addOne() {
        return updater.incrementAndGet(this);
    }
}
```

* CAS 由原本的原子类提供，改为由原子更新器提供；

修改之后：

1. `ct` 字段由 `AtomicInteger` 类变为原始类型 `int`，内存占用从 **堆** 变为 **栈**，原始类型的内存占用少的多；
2. 新增的 `updater` 也是对象，存在于堆上，不过由于是 `static final` 字段，所以系统只会存在一个 `updater` 实例，与 `Counter` 数量没有关系；

所以总体来看，修改后的内存占用要少的多。

### 同时提供普通操作 + 原子操作

有时我们对字段的操作并非 **全部** 是原子操作，甚至 **大部分** 操作是普通操作，只有少数场景才会使用原子操作。

比如 `BufferedInputStream` 中的数组缓存字段，只有 `fill()` 和 `close` 中用到了对数组的 CAS 操作，其他地方都是普通数据操作，如果把该字段设计为 `AtomicReference<byte[]>`，有两个问题：

1. 普通数组操作 **繁琐**，需要不断 `get` 和 `set`；
2. CAS 操作 **耗时** 更多，普通操作替换为 CAS，增加了不必要的耗时；

因此实际上缓存数组被定义为 `volatile byte[]`，并通过 `AtomicReferenceFieldUpdater` 提供 CAS 操作，从而实现普通操作、CAS 的共存和互补。

### 修改其他类的字段

作为应用开发人员，一般不直接使用 `Unsafe`，所以如果要实现 CAS 的字段在第三方类中，则只能使用原子更新器。

## 原子更新器的“缺陷”

还是看下注释：

>Note that the guarantees of the **compareAndSet** method in this class are **weaker** than in other atomic classes. Because this class cannot ensure that all uses of the field are appropriate for purposes of atomic access, it can guarantee atomicity only with respect to other invocations of **compareAndSet** and **set** on the same updater.

即原子更新器的 CAS 比对应的原子类 **弱**，只能保证通过 **同一 updater 实例** 对字段的更新是原子的，以下场景下无法保证原子性：

1. 同时使用多个 updater；
2. 除 updater 外，还使用（无并发处理的）普通更新操作；

>关于第二条，`BufferedInputStream` 中对缓存字段的普通操作都加了 `synchronized` 同步，所以没问题。

---

参考：

1. [AtomicXFieldUpdater，属性原子修改的外部工具类](http://www.wangjialong.cc/2018/01/18/atomicXFieldHelper/)
2. [Real life use and explanation of the AtomicLongFieldUpdate class](https://stackoverflow.com/questions/17239568/real-life-use-and-explanation-of-the-atomiclongfieldupdate-class)
