---
title: 为什么原始包装类和 String 不适合用作内置锁？
date: 2019-08-11 21:33:29
tags:
  - 锁
  - synchronized
categories: Java
---

作为 Java 中最古老的锁机制，内置锁有三种形式：

1. 普通 `synchronized` 方法：内置锁为 **实例对象**；
2. 静态 `synchronized` 方法：内置锁为 **Class 对象**；
3. `synchronized` 块：内置锁为 **指定对象**；

前两个内置锁是 JVM 默认提供的，第三个是用户提供的，一般推荐做法是：

```Java
priviate final Object xxLock = new Object();
```

<!-- more -->

简单粗暴，可以覆盖大部分场景，另外虽然 Java 语法允许 **任何对象** 用作内置锁，但千万不要用原始包装类和 String，原因如下：

* 锁不住
  * 类似 `Integer x = Integer.valueOf(y)` or `String lock = initialLock + sufix` 这样的操作，将返回一个 **新对象**，每次用的都不是同一把锁，自然锁不住；
* 锁过度
  * 由于 **缓存机制** 的存在，貌似不同的原始包装类型 or 字符串实际可能是 **同一对象**，其他任何地方都可以使用该对象做锁，导致锁被 **滥用**，造成 **不必要的串行**； 

## 内置锁实现原理

内置锁的“锁”到底在哪里呢？

一句话答案：锁在对象头的 Mard Word 中！

首先，Java 对象的内存布局分为三部分：

1. 对象头
2. 实例数据
3. 对其填充

其中，实例数据是对象的实际有效数据，包括自己定义的和从父类继承而来的 **字段内容**，同时 JVM 要求对象的 **起始地址** 必须是 8 字节的整数倍，进而对象大小也必须是 8 字节的整数倍，因此若对象头 + 实例数据不是 8 字节的整数倍，需要对其填充。

对象头分为两部分 or 三部分（数组对象），长度为 2 字 or 3 字：

1. Mark Word
2. 类型指针（指向对象的 Class 信息）
3. 数组长度

Mark Word 用于存储 **对象的运行时数据**，如：

1. 对象的 hashCode
2. GC 分代年龄
3. 锁状态信息
4. 线程持有的锁
5. 偏向线程 ID
6. 偏向时间戳
7. ...

Mark Word 存储的信息与对象实际数据 **无关**，为节约存储空间，Mark Word 的数据结构并不固定，它会根据自身状态，变换自己的数据结构，例如：

<img src="/images/java/mark-word.png" alt="Mark Word" style="width: 400px;"/>

32 位机器上的 Mark Word 只有 32 字节，根据 **不同锁状态**，却有 5 种不同数据结构，考虑到 Java 中一切皆对象，这种设计可以大大节约内存空间。

轻量级锁、偏向锁是 Java 1.6 新增的锁优化机制，本文暂不考虑，传统内置锁对应的是“重量级锁”，在重量级锁状态下，Mark Word 存储的是指向操作系统 **互斥量** 的指针，每个线程要获取重量级锁之前，必须先获取互斥量，从而实现多线程的互斥执行。

每个对象都有一个关联的 monitor 对象，monitor 对象是基于操作系统互斥量实现的，因此 **不同对象** 的重量级锁映射到不同信号量。

综上，要实现互斥访问，多个线程必须使用 **同一信号量**，即 **同一 monitor**，即 **同一对象实例**。

包装类型和 String 的问题是：

1. 不同的值，可能对应到同一对象；
2. 相同的值，可能对应到不同对象；

因此，它们不能用作内置锁。

## String

`String` 类内部虽然没有直接使用缓存，但 JVM 会将常用字符串作为常量 **缓存在常量池**（常量池在方法区 or 堆，取决于 JVM 实现）。

### 锁不住

`String` 是不可变类，对它的操作将返回 **新对象**，导致本来想用 **同一把锁** 实现串行的操作变为”各自为政“：

```Java
private static class Worker1 {
    private String lock;
    private String suffix = "-lock";

    public Worker1(String lock) {
        this.lock = lock + suffix;  // 为传入的锁添加后缀
    }

    public void doIt() {
        synchronized (lock) {
            System.out.println("worker1 " + LocalDateTime.now());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

private static class Worker2 {
    private String lock;
    private String suffix = "-lock";

    public Worker2(String lock) {
        this.lock = lock + suffix;  // 为传入的锁添加后缀
    }

    public void doIt() {
        synchronized (lock) {
            System.out.println("worker2 " + LocalDateTime.now());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

使用用”同一把锁“：

```Java
String initialLock = "bank";
Worker1 worker1 = new Worker1(initialLock);
Worker2 worker2 = new Worker2(initialLock);

IntStream.range(0, 3).forEach(ignore -> {
    new Thread(worker1::doIt).start();
    new Thread(worker2::doIt).start();
});
```

本意是 `worker1` 和 `worker2` 都用 `initialLock` 作为锁，这样它们的 `doIt` 方法会串行化，但由于它们内部自作主张修改了传入的锁，导致 `worker1` 和 `worker2` 各自使用 **不同锁**，这样它们的 `doIt` 并行执行：

```
worker1 2019-08-17T11:26:39.935
worker2 2019-08-17T11:26:39.935
worker1 2019-08-17T11:26:40.940
worker2 2019-08-17T11:26:40.941
worker1 2019-08-17T11:26:41.942
worker2 2019-08-17T11:26:41.942
```

### 锁过度

由于字符串可能被缓存在 **常量池**，有时看似毫不相关的两个字符串，实际指向 **同一对象**：

```Java
public class Worker1 {
    private String lock = "lock";  // 锁

    public void doIt() {
        synchronized (lock) {
            System.out.println("worker1 " + LocalDateTime.now());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class Worker2 {
    private String lock = "lock";  // 锁

    public void doIt() {
        synchronized (lock) {
            System.out.println("worker2 " + LocalDateTime.now());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

`Worker1` 和 `Worker2` 中各有一个名为 `lock` 的字符串，但它们的值都是 `"lock"`，看似毫无关系，但实际上，它们是 **同一对象**，它们的 `doIt` 方法是 **串行的**：

```Java
Worker1 worker1 = new Worker1();
Worker2 worker2 = new Worker2();

IntStream.range(0, 3).forEach(ignore -> {
    new Thread(worker1::doIt).start();
    new Thread(worker2::doIt).start();
});
```

输出：

```
worker1 2019-08-17T11:16:37.044
worker2 2019-08-17T11:16:38.050
worker1 2019-08-17T11:16:39.050
worker2 2019-08-17T11:16:40.054
worker1 2019-08-17T11:16:41.059
worker2 2019-08-17T11:16:42.065
```

## 原始包装类型

原始包装类型有：`Byte`、`Character`、`Boolean`、`Short`、`Integer`、`Long`、`Float` 和 `Double`，下面详细分析它们为什么不能用作内置锁。

### Boolean

`Boolean` 主要字段有：

```Java
public final class Boolean implements Serializable, Comparable<Boolean> {
    public static final Boolean TRUE = new Boolean(true);
    public static final Boolean FALSE = new Boolean(false);

    private final boolean value;
    ...
}
```

* `value` 是 `Boolean` 对象包含的实际原始 `boolean` 值；
* `TRUE` 和 `FALSE` 是两个事先创建好的对象，用作 **缓存**；

有两种创建 `Boolean` 的方式：

```Java
// 构造方法，每次创建新对象，不推荐
public Boolean(boolean value) {
    this.value = value;
}

// 工厂方法，返回缓存对象，推荐
public static Boolean valueOf(boolean b) {
    return (b ? TRUE : FALSE);
}
```

构造方法每次都会 **新创建** 一个 `Boolean` 对象，而 `valueOf` 则会复用前面的 `TRUE` 和 `FALSE` 缓存，并且 **赋值** 操作实际使用的也是 `valueOf`：

```Java
// 以下两个等价
Boolean b = false;
Boolean b = Boolean.valueOf(b);
```

假如使用构造函数，则每次创建的都是 **不同对象**，自然不能用作内置锁。

假如使用 `valueOf` or 赋值操作，实际使用的是 **全局缓存**，其他任何线程也可能用 `TRUE` 和 `FALSE` 做锁，锁的使用点有 **无数个**，导致：

1. 无法确定锁用的是否安全，容易发生死锁；
2. 很多地方用同一个锁，造成不必要的串行化，影响效率；

### Character、Byte、Short、Integer、Long

这几个比较类似，放到一起解释，与 `Boolean` 类似，它们都有两种创建方式：构造函数 & `valueOf` 工厂方法，

1. 构造函数：每个创建新对象；
2. `valueOf`：可能使用缓存值；

与 `Boolean.valueOf` 不同的是，这五个类的工厂方法不一定使用缓存值：

```Java
// 必然使用缓存
public static Byte valueOf(byte b) {
    final int offset = 128;
    return ByteCache.cache[(int)b + offset];
}

// [0, 127] 之间的值用缓存
public static Character valueOf(char c) {
    if (c <= 127) { // must cache
        return CharacterCache.cache[(int)c];
    }
    return new Character(c);
}

// [-128, 127] 之间的值用缓存
public static Short valueOf(short s) {
    final int offset = 128;
    int sAsInt = s;
    if (sAsInt >= -128 && sAsInt <= 127) { // must cache
        return ShortCache.cache[sAsInt + offset];
    }
    return new Short(s);
}

// [-128, 127] 之间的值用缓存
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}

// [-128, 127] 之间的值用缓存
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}
```

可以看到，如果值位于 [-128, 127] 之间，则必然使用缓存，否则创建新对象，也不能做内置锁。

### `Float`、`Double`

这两个类的 `valueOf` 直接代理给了构造方法：

```Java
public static Float valueOf(float f) {
    return new Float(f);
}

public static Double valueOf(double d) {
    return new Double(d);
}
```

每次都会返回新对象。

