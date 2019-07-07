---
title: Objects 工具类
tags: Java
categories: Java
date: 2018-05-20 22:46:15
---

Java 7 引入了 `java.util.Objects` 类。

就像 `Arrays` 是 `Array` 的工具类，`Collections` 是 `Collection` 的工具类，`Objects` 即为 `Object` 的工具类，包含 **对象操作** 的方法，其中大部分方法是 **空指针安全** 的。

<!-- more -->

## `requireNonNull` 方法

`requireNonNull` 最初用于方法中的 **参数校验**，其定义如下：

```Java
public static <T> T requireNonNull(T obj) {
  if (obj == null)
    throw new NullPointerException();
  return obj;
}

public static <T> T requireNonNull(T obj, String message) {
  if (obj == null)
    throw new NullPointerException(message);
  return obj;
}

public static <T> T requireNonNull(T obj, Supplier<String> messageSupplier) {
  if (obj == null)
    throw new NullPointerException(messageSupplier.get());
  return obj;
}
```

## `null` 判断

有两个方法用于空指针判断，他们语义恰好相反：

* `isNull`
* `nonNull`

它们实现如下：

```Java
public static boolean isNull(Object obj) {
  return obj == null;
}

public static boolean nonNull(Object obj) {
  return obj != null;
}
```

>`isNull` 和 `nonNull` 是被设计用做 `filter` 的谓语：`filter(Objects::nonNull)`。

## `toString` 方法

```Java
public static String toString(Object o) {
   return String.valueOf(o);
}
```

而 `String.valueOf` 实现如下：

```Java
public static String valueOf(Object obj) {
    return (obj == null) ? "null" : obj.toString();
}
```

因此 `Objects.toString` 对于 `null` 参数，返回 `null` 字符串。

还可以指定参数为 `null` 时的返回值：

```Java
public static String toString(Object o, String nullDefault) {
    return (o != null) ? o.toString() : nullDefault;
}
```

## `equals` 方法

```Java
public static boolean equals(Object a, Object b) {
    return (a == b) || (a != null && a.equals(b));
}
```
* 若 `a` `b` 都是 `null`，则为 `true`；若只有一个为 `null`，则为 `false`

## 其余函数

`Objects` 中还有 `compare` `hash` `hashCode` 等其他函数，暂时没有用到。