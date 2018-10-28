---
title: 源码夜读 | AtomicInteger
date: 2018-10-14 22:47:35
tags:
  - Java
  - AtomicInteger
categories: 源码夜读
---

J.U.C 新增的原子类用 CAS 操作替代锁，更加高效，而且异步，J.U.C 大多数并发类都基于原子类实现，所以理解原子类对于理解整个并发包是非常重要的。

原子类共有 12 个，可以分为以下 4 类：

* 标量类（`AtomicInteger`、`AtomicLong`、`AtomicReference`、`AtomicBoolean`）
* 数组类（`AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray`）
* 更新器类（`AtomicIntegerFieldUpdater`、`AtomicLongFieldUpdater`、`AtomicReferenceFieldUpdater`）
* 复合变量类（`AtomicMarkableReference`、`AtomicStampedReference`）

<!-- more -->

>实际上，atomic 包中还有另外 5 个类：
>* `DoubleAccumulator`
>* `DoubleAdder`
>* `LongAccumulator`
>* `LongAdder`
>* `Striped64`

对于 4 个标量类而言，实现基本类似，因此以 `AtomicInteger` 来剖析其实现。

## 继承关系

`AtomicInteger` 的继承关系非常简单：

```Java
public class AtomicInteger extends Number implements java.io.Serializable
```

`Number` 是代表数字的抽象类：

```Java
public abstract class Number implements java.io.Serializable
```

常见的基本数字类型的包装类都是 `Number` 的子类，例如 `Short`、`Integer`、`Long`、`Double`、`Byte` 等，`AtomicInteger` 也是数字，因此继承 `Number` 合情合理。

可以把 `Number` 代表的数值转换为任意基本数字类型：

```Java
public abstract int intValue();
public abstract long longValue();
public abstract float floatValue();
public abstract double doubleValue();
public byte byteValue() {
    return (byte)intValue();
}
public short shortValue() {
    return (short)intValue();
}
```

* 转换过程中，可能出现精度损失，或截断；

## 内部状态

`AtomicInteger` 的内部状态非常简单，只有 3 个字段：

```Java
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

private volatile int value;
```

其中 `unsafe` 和 `valueOffset` 是不可变字段，`value` 是可变状态，并通过 `volatile` 保证 `value` 字段的内存可见性和顺序性。

因为 `valueOffset` 是 `static` 变量，因此在 static initializer 中初始化，并在在 `AtomicInteger` 类加载时初始化：

```Java
static {
  try {
    valueOffset = unsafe.objectFieldOffset
    (AtomicInteger.class.getDeclaredField("value"));
  } catch (Exception ex) { throw new Error(ex); }
}
```

## 关键方法

`AtomicInteger` 的初衷就是在不使用锁的前提下，实现原子的读-改-写操作，这是通过 `Unsafe` 类提供的 CAS 操作实现的，CAS 操作有底层 CPU 直接支持。

因此 CAS 是 `AtomicInteger` 的重点，不过下面我们会解析 `AtomicInteger` 的全部方法。

### 构造

`AtomicInteger` 提供两个构造函数，既可以用给定值初始化 `AtomicInteger`：

```Java
public AtomicInteger(int initialValue) {
    value = initialValue;
}
```

也可用无参构造，此时初始值默认为 0：

```Java
public AtomicInteger() {
}
```

### 获取

怎么获取 `AtomicInteger` 中封装的 int 值呢？

最基本的有：

```Java
public final int get() {
    return value;
}
```

也可以返回其他类型的数值，例如 `intValue`、`longValue`、`floatValue` 和 `doubleValue`，这些都是 `Number` 类定义的方法，用于数值类之间的相互转换。

### 设置

#### `set`

既可以直接设置 `value` 的值：

```Java
public final void set(int newValue) {
    value = newValue;
}
```

由于 `value` 是 `volatile` 变量，通过内存屏障，`set` 对 `value` 的修改对其他线程是 **立即可见** 的，所以无需添加 `synchronized`。

#### `lazySet`

大部分场景直接用 `set` 即可，但 `set` 内存屏障将禁止 **重排序**，这会带来一定的性能消耗，因此非常关心性能，同时不需要 **立即看到** `set` 的效果时，可以使用 `lazySet`：

```Java
// Eventually sets to the given value.
public final void lazySet(int newValue) {
    unsafe.putOrderedInt(this, valueOffset, newValue);
}
```

`lazySet` 不保证设置的结果立即对其他线程可见，相比 `set` 有一定的性能优势。

#### `getAndSet`

若想设置新值，同时返回旧值，应该如何实现呢？

因为 `value` 是 `volatile` 变量，所以对 `value` 的 read/write 具备 happens-before 关系，所以一个很容易想到的实现为：

```Java
AtomicInteger counter = new AtomicInteger();
int oldV = counter.get();
counter.set(10);
```

但很不幸，该实现是错的，因为 `counter.get()` 与 `counter.set(10)` 之间可能插入其他线程的 `set`，所以 `oldV` 不能保证是 `set(10)` 执行时的 `value` 值，当然通过锁将 `get` 与 `set(10)` 变成原子操作可以满足需求，但我们使用 `AtomicInteger` 就是为了避免使用锁，所以也不能这么做。

解决办法是 `getAndSet`：

```Java
/**
 * Atomically sets to the given value and returns the old value.
 */
public final int getAndSet(int newValue) {
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}
```

关键词是 `atomically`，即将 `set` 和 `get` 合并成一个原子操作，同时避免使用锁，依旧借助 `unsafe` 实现。

### 算术操作

整数常见的算术操作有：

* 自增、自减
* 加减

在 `AtomicInteger` 中他们都是基于 `Unsafe.getAndAddInt` 实现的，并且都是 **原子操作**。

#### 自增、自减

自增、自减实现：

```Java
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

public final int getAndDecrement() {
    return unsafe.getAndAddInt(this, valueOffset, -1);
}

public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

以上方法分别实现了 `i++`、`++i`、`i--`、`--i`，它们都是基于 `Unsafe.getAndAddInt` 实现的，该方法实现 `value` 加操作，且返回旧值。

#### 加减

任意值的加、减法实现：

```Java
/**
 * Atomically adds the given value to the current value.
 */
public final int getAndAdd(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta);
}

/**
 * Atomically adds the given value to the current value.
 */
public final int addAndGet(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}
```

注意并没有 `getAndSubtract`，因为加负值即可实现减法，所以 `getAndAdd` 足以。

### CAS

#### `compareAndSet`

`getAndSet` 无脑地设置 `value` 的值，但实际场景不可能一直如此简单，有时要求 `value` 满足特定条件时才设置，这是非常典型的原子复合操作：

1. 检查某条件是否成立；
2. 根据条件成功、失败执行不同操作；

在业务代码中，该操作一般用锁实现，但 `AtomicInteger` 原生提供了该操作：

```Java
/**
 * Atomically sets the value to the given updated value if the current value {@code ==} the expected value.
 */
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

`compareAndSet` 非常有用，`AtomicInteger` 中很多方法是基于它实现的，只有 `value` 的当前值等于 `expect` 时，才把 `value` 设置为 `update`，同时如果设置成功则返回 `true`，否则返回 `false`。

借助返回值可以检测方法的执行结果，因此可以在循环操作中不断执行 `compareAndSet`，直到成功，后面会看到，很多方法都是这种实现套路。

#### `weakCompareAndSet`

`weakCompareAndSet` 是 `compareAndSet` 的弱化版本：

```Java
/**
 * Atomically sets the value to the given updated value if the current value {@code ==} the expected value.
 *
 * weakCompareAndSet May fail spuriously and does not provide ordering guarantees</a>, so is
 * only rarely an appropriate alternative to compareAndSet.
 */
public final boolean weakCompareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

`weakCompareAndSet` 使用场景很少，在某些平台上它的效率可能比 `compareAndSet` 高，但有如下两点限制：

* may fail spuriously
* not provide ordering guarantees

### 基于 CAS 的方法

`AtomicInteger` 有很多方法基于 `compareAndSet` 方法实现，我们把这些方法放到一起分析。

`compareAndSet` 一般放在一个 `do-while` 循环中使用：

```Java
int oldValue, newValue;
do {
    oldValue = get();
    newValue = f(xxx)
} while (!compareAndSet(oldValue, newValue))
```

旧值一般直接用 `get` 获取，而新值则可以通过任意逻辑计算而来，因为 `f` 在 `while` 循环中可能被 **不断重试**，所以 `f` 必须是 **纯函数**，不能有任何副作用。

#### 一元函数

除了设置（`getAndSet`）、自增（`getAndIncrement`）、自减（`getAndDecrement`）、加减（`getAndAdd`）外，`AtomicInteger` 还有更加灵活的更新方式：

```Java
/**
 * Atomically updates the current value with the results of
 * applying the given function, returning the previous value. The
 * function should be side-effect-free, since it may be re-applied
 * when attempted updates fail due to contention among threads.
 *
 * @param updateFunction a side-effect-free function
 * @return the previous value
 * @since 1.8
 */
public final int getAndUpdate(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return prev;
}

public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return next;
}
```

* `getAndUpdate` 与 `updateAndGet` 实现几乎相同，区别仅仅是返回旧值还是新值；
* `IntUnaryOperator` 必须是纯函数；

其中 `IntUnaryOperator` 类型为 `Int => Int`，即一个用于 `Int` 的一元操作符：

```Java
@FunctionalInterface
public interface IntUnaryOperator {
    int applyAsInt(int operand);

    default IntUnaryOperator compose(IntUnaryOperator before) {
        Objects.requireNonNull(before);
        return (int v) -> applyAsInt(before.applyAsInt(v));
    }

    default IntUnaryOperator andThen(IntUnaryOperator after) {
        Objects.requireNonNull(after);
        return (int t) -> after.applyAsInt(applyAsInt(t));
    }

    static IntUnaryOperator identity() {
        return t -> t;
    }
}
```

参考 Scala 的 `apply` 方法，`IntUnaryOperator` 的抽象方法为 `applyAsInt`。

#### 二元函数

前面的 `getAndUpdate` 只能通过一元操作符更新 `value`，有时我们需要联合 `value` 与其他外部世界的值来计算 `value` 的新值，此时可用 `getAndAccumulate`：

```Java
/**
 * Atomically updates the current value with the results of
 * applying the given function to the current and given values,
 * returning the previous value. The function should be
 * side-effect-free, since it may be re-applied when attempted
 * updates fail due to contention among threads.  The function
 * is applied with the current value as its first argument,
 * and the given update as the second argument.
 *
 * @param x the update value
 * @param accumulatorFunction a side-effect-free function of two arguments
 * @return the previous value
 * @since 1.8
 */
public final int getAndAccumulate(int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));
    return prev;
}

public final int accumulateAndGet(int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));
    return next;
}
```

与 `getAndUpdate` 几乎相同，区别仅仅是把一元操作符换成了二元操作符，并增加了一个 `int` 参数。

唯一值得注意的是 `IntBinaryOperator` 的参数顺序：

* 以 `value` 当前值为第一个参数；
* 以 `x` 为第二个参数；

### 其他

到此为止，`AtomicInteger` 主要方法我们都已经见过了，剩下的是一些非常常规的方法：

```Java
public String toString() {
    return Integer.toString(get());
}

public int intValue() {
    return get();
}

public long longValue() {
    return (long)get();
}

public float floatValue() {
    return (float)get();
}

 public double doubleValue() {
    return (double)get();
}
```

## 总结

`AtomicIntger` 的关键是 `compareAndSet` 方法，基于它可实现非常灵活的 **无锁** 算法。
