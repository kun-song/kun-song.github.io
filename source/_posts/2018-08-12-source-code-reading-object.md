---
title: 源码分析 | Object
date: 2018-08-12 19:21:55
tags:
  - Java
  - Object
categories: 源码分析
---

`Object` 源码非常简单，只有区区几个方法，除 `registerNatives`、`getClass` 外，其余方法可分为两类：

* 子类覆盖
  + `hashCode`
  + `equals`
  + `clone`
  + `toString`
  + `finalize`
* 并发控制
  + `notify`
  + `notifyAll`
  + `wait` 3 个重载版本

虽然源码很简单，但要准确理解它们的含义却没那么容易，下面介绍这几个方法的用途，以及惯用法。

<!-- more -->

## 子类覆盖

`equals`、`hashCode`、`clone`、`toString` 和 `finalize` 5 个方法设计为专门被子类覆盖，且对于如何实现它们，Java 有明确的约定（general contracts），JDK 部分类（如 `HashMap` 和 `HashSet`）的功能依赖这些约定，子类实现时必须遵守它们，否则可能出现功能异常。

### `equals`

`equals` 用于判断本对象与 `obj` 对象是否“相等”，默认实现如下：

```Java
public boolean equals(Object obj) {
    return (this == obj);
}
```

* `==` 比较引用的内存地址；

默认只有当 `obj` 与 `this` 指向 **同一实例** 时 `equals` 结果才是 `true`，即实例 `A` 只与自己“相等”，其他所有场景都不相等。

#### 何时不覆盖 `equals` ？

根据《Effective Java》：

1. Each instance of the class is **inherently unique**.

若每个实例本质上各不相同，例如每个 `Thread` 实例都代表一个活跃的调度实体，此时每个实例只能与自己“相等”，`Object.equals` 默认实现非常适合。

相对而言，`Integer` 实例代表的是“值”，此时默认实现明显不合适，因为代表同一值的两个 `Integer` 实例逻辑上应该相等，此时需要覆盖默认 `equals` 实现。

2. There is **no need** for the class to provide a "logical equality" test.

`equals` 作用是提供逻辑上“相等”的概念，如果根本不需要这种概念，那当然没有必要覆盖 `euqals`，实际上，此时根本不会使用 `equals`。

例如 `java.util.regex.Pattern` 设计者认为用户根本不需要判断两个 `Pattern` 实例是否相等，此时自然不需要多此一举去实现 `equals`。

3. A **superclass** has already overridden `equals`, and the superclass behavior is appropriate for this class.

大部分 `Set`、`List` 和 `Map` 实现继承了 `AbstractSet`、`AbstractList` 和 `AbstractMap` 的 `equals` 实现。

4. The class is private or package-private, and you are certain that its `equals` method will **never be invoked**.

若 `equals` 永远不会被调用，当然无需覆盖了，如果不放心，还可以覆盖 `equals` 使其抛出异常，以避免误用：

```Java
public boolean equals(Object obj) {
    throw new AssertionError();
}
```

#### 何时覆盖 `equals` ？

当类满足两个条件时，一般需要覆盖默认 `equals`：

* 该类具有逻辑上的“相等”概念，且与默认 `equals` 实现的相等性不同；
* 其父类没有覆盖 `equals`；

值类（value class）一般满足以上两点，如 `String` 和 `Integer`，对于值类而言，“相等”的不仅仅是同一个对象，只要两个对象表示的 **值** 相等，这两个对象在逻辑上就是相等的。

值类必须覆盖 `equals` 方法才能用做 `Map` 的键，才能存储于 `Set`。

#### 约定



### `hashCode`


### `clone`


### `toString`


### `finalize`


## 并发控制


### `wait`


### `notify`


### `notifyAll`

## `registerNatives` & `getClass`

### `registerNatives`

`registerNatives` 用于注册 native 方法，且在 `static` 块中调用，因此 **类加载** 时就会执行 `registerNatives` 方法：

```Java
private static native void registerNatives();
static {
    registerNatives();
}
```

### `getClass`

`getClass` 为 `final` 方法，不能覆盖，用于在运行时获取当前对象的 `Class` 对象：

```Java
public final native Class<?> getClass();
```

## 常见问题

### `Object` 是否有父类？


## 源码

```Java
public class Object {

    private static native void registerNatives();
    static {
        registerNatives();
    }

    public final native Class<?> getClass();

    public native int hashCode();

    public boolean equals(Object obj) {
        return (this == obj);
    }

    protected native Object clone() throws CloneNotSupportedException;

    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }

    public final native void notify();

    public final native void notifyAll();

    public final native void wait(long timeout) throws InterruptedException;

    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos > 0) {
            timeout++;
        }

        wait(timeout);
    }

    public final void wait() throws InterruptedException {
        wait(0);
    }

    protected void finalize() throws Throwable { }
}
```
