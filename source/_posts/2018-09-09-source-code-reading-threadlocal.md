---
title: 源码夜读 | ThreadLocal
date: 2018-09-02 22:23:16
tags:
  - Java
  - ThreadLocal
categories: 源码夜读
---

当状态变量 **非线程安全** 时，是无法在并发场景中直接共享的，常见的解决办法有：

* 同步（`synchronized` 或 `Lock`）
* 修改为线程安全的实现
* 线程封闭

其中第三种方法有两种实现方式：

* 将对共享状态的读写操作集中到 **一个线程** 中，即在事实上将该变量 **封闭** 到单线程；
* `ThreadLocal`；

<!-- more -->

`ThreadLocal` 即线程本地变量，它可以每个线程都有共享变量的一个 **副本**，各个线程无法感知其他线程的中线程本地变量的值。当线程终止后，属于该线程的变量副本将被 GC 回收。

`ThreadLocal` 实例一般声明为 `private static` 字段。

## 字段

`ThreadLocal` 有 3 个字段：

```Java
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

其中 `threadLocalHashCode` 字段最为重要。每个线程都有一个 `Thread.threadLocals`（类型为 `ThreadLocalMap`），该 map 以 `ThreadLocal` 实例为 key，以本地变量为 value，`threadLocalHashCode` 是专门为 `ThreadLocalMap` 优化过的，可以减少哈希碰撞，注意 `threadLocalHashCode` 用于普通 hash map 效果并不好。

`threadLocalHashCode` 初始值为 0，之后每次增加 `0x61c88647`，至于为什么是这个神奇的数字，后面解析 `ThreadLocalMap` 时再再详细解释。

## 初始化

第一种是构造函数 + `set` 方法：

```Java
ThreadLocal<String> x = new ThreadLocal<>();
x.set("Hello");
```

第二种是匿名内部类：

```Java
ThreadLocal<String> x = new ThreadLocal<String>() {
    @Override
    protected String initialValue() {
        return "hello";
    }
};
```

第三种是借助 `withInitial` 静态方法：

```Java
ThreadLocal<String> x = ThreadLocal.withInitial(() -> "Hello"); 
```

匿名内部类很冗长，已经不推荐使用，推荐使用 `withInitial` 进行初始化。

## `get`、`set`、`remove`

### `set`

```Java
/**
  * Sets the current thread's copy of this thread-local variable
  * to the specified value.  Most subclasses will have no need to
  * override this method, relying solely on the {@link #initialValue}
  * method to set the values of thread-locals.
  *
  * @param value the value to be stored in the current thread's copy of
  *        this thread-local.
  */
public void set(T value) {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}
```

首先获取当前线程的引用，然后通过 `getMap` 获取当前线程的 `ThreadLocalMap`，`getMap` 实现很简单：

```Java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

其中 `threadLocals` 是 `Thread` 类的一个字段，且默认值为 `null`：

```Java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

>线程之间无法访问彼此的 `threadLocals` 变量，因此可以保证每个 `ThreadLocal` 实例只在 `threadLocals` 中出现一次，从而保证线程私有。

若当前线程的 `ThreadLocalMap` 不为空，则以当前 `ThreadLocal` 实例为 key，以参数值 `value` 为 value，将该 k-v 存入线程的 `ThreadLocalMap` 对象中：

```Java
map.set(this, value);
```

若当前线程的 `ThreadLocalMap` 为空，则为当前线程创建一个新的 `ThreadLocalMap` 值，`createMap` 源码为：

```Java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

因为 `Thread` 和 `ThreadLocal` 在同一个 package 内，且 `threadLocals` 访问级别为 `package`，因此 `ThreadLocal.createMap` 可以直接设置 `threadLocals` 的值。

### `get`

```Java
/**
  * Returns the value in the current thread's copy of this
  * thread-local variable.  If the variable has no value for the
  * current thread, it is first initialized to the value returned
  * by an invocation of the {@link #initialValue} method.
  *
  * @return the current thread's value of this thread-local
  */
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}
```

首先在当前线程的 `ThreadLocalMap` 中查询，若查询成功，则返回结果；若当前线程的 `threadLocals` 为 `null` 或查询结果为 `null`，则调用 `setInitialValue`，并将其结果作为返回值，`setInitialValue` 源码如下：

```Java
/**
  * Variant of set() to establish initialValue. Used instead
  * of set() in case user has overridden the set() method.
  *
  * @return the initial value
  */
private T setInitialValue() {
  T value = initialValue();
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
  return value;
}
```

首先通过 `initialValue` 获取本地变量的初始值：

```Java
/**
  * Returns the current thread's "initial value" for this
  * thread-local variable.  This method will be invoked the first
  * time a thread accesses the variable with the {@link #get}
  * method, unless the thread previously invoked the {@link #set}
  * method, in which case the {@code initialValue} method will not
  * be invoked for the thread.  Normally, this method is invoked at
  * most once per thread, but it may be invoked again in case of
  * subsequent invocations of {@link #remove} followed by {@link #get}.
  *
  * <p>This implementation simply returns {@code null}; if the
  * programmer desires thread-local variables to have an initial
  * value other than {@code null}, {@code ThreadLocal} must be
  * subclassed, and this method overridden.  Typically, an
  * anonymous inner class will be used.
  *
  * @return the initial value for this thread-local
  */
protected T initialValue() {
  return null;
}
```

从源码可以看出，`initialValue` 是 `protected` 方法，子类需要覆盖（参考匿名内部类初始化），若没有调用过 `set` 方法，则 `get` 将借助该方法获取本地变量的初始值，并将其存入 `threadLocals`。

### `remove`

```Java
/**
  * Removes the current thread's value for this thread-local
  * variable.  If this thread-local variable is subsequently
  * {@linkplain #get read} by the current thread, its value will be
  * reinitialized by invoking its {@link #initialValue} method,
  * unless its value is {@linkplain #set set} by the current thread
  * in the interim.  This may result in multiple invocations of the
  * {@code initialValue} method in the current thread.
  *
  * @since 1.5
  */
public void remove() {
  ThreadLocalMap m = getMap(Thread.currentThread());
  if (m != null)
    m.remove(this);
}
```

`remove` 将 `ThreadLocal` 实例从 `threadLocals` 中删除，若删除后再 `get` 将导致本地变量 **重新初始化**，从而丢失之前的状态。

## `ThreadLocalMap`

`ThreadLocalMap` 一个专门为 `ThreadLocal` key 定制实现的 hash map，它的所有方法只能在 `ThreadLocal` 类内部使用，其访问级别为 `package`，因此同一个包中的 `Thread` 可以声明一个 `ThreadLocalMap` 字段（`t.threadLocals`）。

### 字段

`ThreadLocalMap` 字段如下：

```Java
/**
  * The initial capacity -- MUST be a power of two.
  */
private static final int INITIAL_CAPACITY = 16;

/**
  * The table, resized as necessary.
  * table.length MUST always be a power of two.
  */
private Entry[] table;

/**
  * The number of entries in the table.
  */
private int size = 0;

/**
  * The next size value at which to resize.
  */
private int threshold; // Default to 0
```

可以看到 `ThreadLocalMap` 内部使用一个 `Entry` 数组存储，其初始值为 16，且容量必须是 2 的指数。

## `Entry`

`Entry` 定义如下：

```Java
/**
  * The entries in this hash map extend WeakReference, using
  * its main ref field as the key (which is always a
  * ThreadLocal object).  Note that null keys (i.e. entry.get()
  * == null) mean that the key is no longer referenced, so the
  * entry can be expunged from table.  Such entries are referred to
  * as "stale entries" in the code that follows.
  */
static class Entry extends WeakReference<ThreadLocal<?>> {
  /** The value associated with this ThreadLocal. */
  Object value;

  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}
```

`Entry` 继承 `WeakReference`，以 `ThreadLocal` 实例为 key，以 `Object` 为 value。当 value 为 `null` 时，表示对应的 key 不再使用，因此可以从 `table` 中删除，该类 entry 被称为 "stale entry"。

>继承 `WeakReference` 是为了避免内存泄漏，当线程结束后，`threadLocals` 中的 entry 不再引用，变成不可达对象，GC 之。

### 构造

一般用 k-v 映射构造：

```Java
/**
  * Construct a new map initially containing (firstKey, firstValue).
  * ThreadLocalMaps are constructed lazily, so we only create
  * one when we have at least one entry to put in it.
  */
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  table = new Entry[INITIAL_CAPACITY];
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  table[i] = new Entry(firstKey, firstValue);
  size = 1;
  setThreshold(INITIAL_CAPACITY);
}
```

该 k-v 在 `Entry` 数组中的索引由 `firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1)` 计算得来，前面说过 `ThreadLocal.threadLocalHashCode` 每次增加 0x61c88647，目的是使此处计算出的索引尽可能 **均匀分布**，从而减少哈希碰撞。

另外 `hashCode & (size - 1)` 功能与 `hashCode % size` 相同，相当于取余，但更加高效，`HashMap` 也使用了该技巧。

还可以从已有的 `ThreadLocalMap` 构造：

```Java
/**
  * Construct a new map including all Inheritable ThreadLocals
  * from given parent map. Called only by createInheritedMap.
  *
  * @param parentMap the map associated with parent thread.
  */
private ThreadLocalMap(ThreadLocalMap parentMap) {
  Entry[] parentTable = parentMap.table;
  int len = parentTable.length;
  setThreshold(len);
  table = new Entry[len];

  for (int j = 0; j < len; j++) {
    Entry e = parentTable[j];
    if (e != null) {
      @SuppressWarnings("unchecked")
      ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
      if (key != null) {
        Object value = key.childValue(e.value);
        Entry c = new Entry(key, value);
        int h = key.threadLocalHashCode & (len - 1);
        while (table[h] != null)
          h = nextIndex(h, len);
        table[h] = c;
        size++;
      }
    }
  }
}
```

注意该构造方式只能从 `Thread.createInheritedMap` 调用！

### `getEntry`

```Java
/**
  * Get the entry associated with key.  This method
  * itself handles only the fast path: a direct hit of existing
  * key. It otherwise relays to getEntryAfterMiss.  This is
  * designed to maximize performance for direct hits, in part
  * by making this method readily inlinable.
  *
  * @param  key the thread local object
  * @return the entry associated with key, or null if no such
  */
private Entry getEntry(ThreadLocal<?> key) {
  int i = key.threadLocalHashCode & (table.length - 1);
  Entry e = table[i];
  if (e != null && e.get() == key)
    return e;
  else
    return getEntryAfterMiss(key, i, e);
}
```

`getEntry` 相当于其他 map 的 `get`，该方法直接去 key 的索引位置查找，找到则直接返回，否则通过 `getEntryAfterMiss` 从当前位置继续查找：

```Java
/**
  * Version of getEntry method for use when key is not found in
  * its direct hash slot.
  *
  * @param  key the thread local object
  * @param  i the table index for key's hash code
  * @param  e the entry at table[i]
  * @return the entry associated with key, or null if no such
  */
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
  Entry[] tab = table;
  int len = tab.length;

  while (e != null) {
    ThreadLocal<?> k = e.get();
    if (k == key)
      return e;
    if (k == null)
      expungeStaleEntry(i);
    else
      i = nextIndex(i, len);
    e = tab[i];
  }
  return null;
}
```

`getEntryAfterMiss` 将从 `Entry` 数组的 `i` 位置开始，依次查看各个 `Entry`，若 `Entry` 的 key 为指定查找的键，则直接返回，否则从下个索引继续查找，为避免索引溢出有效 `Entry` 列表，`nextIndex` 对 `len` 取模：

```Java
/**
  * Increment i modulo len.
  */
private static int nextIndex(int i, int len) {
  return ((i + 1 < len) ? i + 1 : 0);
}
```

注意，若当前 `Entry` 的 key 为 `null`，则删除。

### `set`

// TODO

## 使用场景

除实现线程封闭外，`ThreadLocal` 可以简化变量访问，例如若变量 `a` 在线程声明周期中被使用多次，且使用地点分布在不同方法中，此时参数传递比较繁琐，借助 `ThreadLocal` 可以在任何地方方便地访问 `a`，更加灵活。
