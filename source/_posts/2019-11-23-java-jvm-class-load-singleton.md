---
title: 类加载（三）：类加载与单例模式
date: 2019-11-23 16:41:15
tags:
  - 类加载
  - 单例模式
categories: JVM
---

借助类加载（中的“初始化”阶段）时机 + 静态内部类，可实现 **线程安全** 的、**延迟初始化** 的单例模式，并发控制借助 JVM 内置实现，因此并发 *性能* 非常好。

<!-- more -->

## 基础理论

1. JVM 实现保证，类的“初始化”阶段是 **线程安全** 的，不同线程可能同时触发同一类的初始化，但最终“初始化”仅会执行一次，其余初始化实际获取第一个初始化的结果；
2. JVM 规范保证只有访问类的 **静态方法** 时，才会触发该类的初始化；
3. 单例模式要求 **私有** 的构造函数；

## 线程安全 + 非延迟初始化

由于访问 **静态字段** 时才会执行类的初始化，很容易写出：

```Java
public class Singleton {
    private static Singleton INSTANCE = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

类加载与创建实例并不等价，但 `INSTANCE` 是一个静态字段，它的初始化表达式会被 Javac 编译到 `<clinit>` 方法中，该方法会在类加载的“初始化”阶段执行，这里的初始化表达式恰好是 **创建实例**，因此“初始化”阶段恰好会创建一个 `INSTANCE` 实例，单例、线程安全由 JVM 类加载机制保证：

* 单例：类加载 **仅执行一次**，所以 `<clinit>`（`new Singleton()`）必然只被执行一次，从而保证全局 **只有一个** `Singleton` 实例；
* 线程安全：JVM 保证同一个类的加载过程是 **串行** 的，第一个加载成功后，剩余加载直接返回其结果，不会重复加载，因此 `getInstance` 无需 `synchronized` 修饰，天然是线程安全的；

咋一看只有执行 `getInstance` 方法时，才会触发加载 `Singleton`，进而在“初始化”阶段执行 `<clinit>` 方法，进而执行 `new Singleton()` 创建实例，似乎很完美，是延迟初始化的。

问题在于 `Singleton` 类的类加载不完全受 `getInstance` 方法调用的控制，任何导致 `Singleton` 类加载的场景都会触发创建 `Singleton` 类的实例，我们要求只有调用 `getInstance` 时才创建实例，因此该方式非延迟初始化。

## 线程安全 + 延迟初始化

前面实现非延迟初始化的根源在于 `Singleton` 类是公共类，有很多方式会触发其类加载（类加载中的“初始化”阶段），可通过 **私有的静态内部类** 规避该问题：

```Java
public class Singleton {
    private static class LazyHolder {
        static Singleton INSTANCE = new Singleton();
    }

    private Singleton() {}

    public static Singleton getInstance() {
        return LazyHolder.INSTANCE;
    }
}
```

`LazyHolder` 是 **私有类**，除 `Singleton` 之外，其他类无法访问，也就无法触发其初始化，**唯一** 触发 `LazyHolder` 初始化的方式是：

1. 调用 `getInstance`；
2. 间接访问 `LazyHolder.INSTANCE`，即访问 `LazyHolder` 的静态字段；
3. 导致 `LazyHolder` 执行“初始化”阶段，即执行其 `<clinit>` 方法；
4. `<clinit>` 包含 `static Singleton INSTANCE = new Singleton();`，通过 `new` 指令触发 `Singleton` 的“初始化”阶段；

关键在于即使执行 `Singleton` 的类加载（加载、连接、初始化），最多执行该类的 `<clinit>` 方法，但现在 `Singleton` 没有任何 **静态字段**，所以它的 `<clinit>` 基本是空的，很快就能执行完，也不会创建 `Singleton` 实例。

## 总结

* 类加载 = 不用 `synchronized` 实现线程安全 + 天然单例
* 静态内部类 = 延迟初始化

---

参考：

* [Initialization-on-demand holder idiom（好）](https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom)
* [Fastest Thread-safe Singleton in Java](http://literatejava.com/jvm/fastest-threadsafe-singleton-jvm/)
