---
title: ThreadLocal 之过期删除、内存泄漏
date: 2019-10-27 09:54:09
tags:
  - ThreadLocal
categories: Java
---

既然 `ThreadLocal` 用以实现线程本地变量，那线程终止时，需要删除这些过期数据，否则会发生内存泄漏。

<!-- more -->

## 过期删除

删除过期数据分为 3 种场景：

1. 显式删除（推荐用法）
   1. 调用 `ThreadLocal.remove()`；
2. 隐式删除
   1. 删除 key：弱引用指向的 `ThreadLocal` 实例可自动回收，从 k-v 变成 `null`-v；
   2. 删除 value：`ThreadLocalMap` 使用 `expungeStaleEntry` 清除 `null`-v；
3. 自动删除
   1. 线程终止时会执行 `exit` 方法，其中有删除逻辑；

显式删除不会发生内存泄漏，而隐式删除有可能。

### 显式删除

在 **线程池** 场景中，线程会一直运行，除非异常终止，此时推荐任务执行结束后显式删除“本地变量”：

```Java
ThreadLocal<String> x = new ThreadLocal();
try {
    x.set("hello");
    ...
} finally {
    localName.remove();
}
```

原因有两个：

* 若不显式删除，则下次任务执行时可能读取到上次设置的值，即 **脏读**；
* 线程一直处于运行状态，若依赖隐式删除，可能发生 **内存泄漏**；

### 隐式删除

隐式删除即不显式调用 `remove`，不考虑任何删除操作，只管 `get/set`，得益于 `ThreadLocal` 的优良设计，一般也没啥问题，但极端情况下可能发生 **内存泄漏**。

隐式删除需要分别删除 k-v 映射中的 k 和 v：

1. 删除 k：弱引用做 key，GC 自动回收 k；
2. 删除 v：`get/set` 底层会调用 `expungeStaleEntry` 方法删除 `null`-v 映射中的 v；

#### 删除 k

`ThreadLocalMap` 的元素类型为 `Entry`：

```Java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) { super(k); value = v; }
}
```

它是一个 **弱引用**（[Java 四大引用类型](http://songkun.me/2019/10/19/2019-10-19-java-jvm-4-kind-reference/)），只被弱引用指向的对象将在下次 GC 时被回收。

`Entry` 指向 `ThreadLocal` 实例，因此若只有 `Thread.threadlocals` 中的某个元素指向 `x`，则 `x` 只能存活到下次 GC 之前：

```Java
ThreadLocal<Student> x = ThreadLocal.withInitial(() -> new Student());
```

不过 `Thread.threadlocals` 是一个 map，里面存储的是 k-v，`x` 仅仅是其中的 key，即使 key 变成 `null`，`threadlocals` 中还是会保存该 k-v。

#### 删除 v

得益于弱引用，待清除的 k-v 映射的 key 会自动被 GC 回收，从 k-v 变成 `null`-v，`ThreadLocalMap` 很多方法会调用 `expungeStaleEntry` 清除这些 key 为 `null` 的映射，从而完成过期数据的清除。

因此只要线程 t 还有未被 GC 的 `ThreadLocal` 实例，当调用该实例的 `get/set` 时，`t.threadLocals` 中未被清除的 `null`-v 映射都会被删除。

### 自动删除

`Thread.exit()` 方法会清理当前线程的所有状态字段：

```Java
private void exit() {
    if (group != null) {
        group.threadTerminated(this);
        group = null;
    }
    /* Aggressively null out all reference fields: see bug 4006245 */
    target = null;
    /* Speed the release of some of these resources */
    threadLocals = null;
    inheritableThreadLocals = null;
    inheritedAccessControlContext = null;
    blocker = null;
    uncaughtExceptionHandler = null;
}
```

其中包括 `threadLocals` 哈希表，它变为 `null` 后，里面的 k-v 若无其他引用路径，则被 GC 回收。

## 内存泄漏

### 泄漏原因

前面说过隐式删除在极端场景下可能发生内存泄漏，该极端场景为：

1. 线程 t 一直运行，不终止（线程池）；
2. `null`-v：`t.threadLocals` 中某个 k（`ThreadLocal` 实例）无其他引用，只有此处的弱引用；
3. 线程 t 无 **其他** 线程本地变量操作 `get/set`；

以上 3 点任何一点不满足，都不会发生内存泄漏，例如即使在线程池场景下出现 `null`-v 映射，但若线程 t 操作过其他 `ThreadLocal.get/set`，也会顺便把该 `null`-v 删除。

其实逻辑很简单：

1. 线程不终止 -> `t.threadLocals` 不为 `null`，从而引用其保存的 k-v；
2. 弱引用 k 被回收 -> k-v 变成 `null`-v；

以上两点导致 `null`-v 中的 v 只被 `t.threadLocals` 引用，同时由于 k 为 `null`，外部又无法真正获取 v 从而使用它，也无法执行 `k.remove()` 进行删除，即发生内存泄漏。

### 如何避免

理论上，破坏 3 点中的任何一点都可以避免内存泄漏；实际上，第 1、3 两点很难控制，消除第 2 点比较现实。

`ThreadLocal` 注释有一句：

>ThreadLocal instances are typically private **static** fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).

即推荐将其声明为 `static` 字段，这样 `t.threadLocals` 中的 key 会永远有一个外部 **强引用**，永远不会出现 `null`-v，任何时候都可以通过 `ThreadLocal.remove()` 删除该 k-v，从而避免内存泄漏。

## 总结

1. 推荐使用 `ThreadLocal.remove()` 显式删除；
2. 长时间运行线程 + `null`-v + 不再执行 `ThreadLocal.get/set` 会造成内存泄漏；
3. 推荐将 `ThreadLocal` 声明为 `static` 字段，避免 `null`-v 出现，以便任何是否可通过 `x.remove()` 将其删除；
