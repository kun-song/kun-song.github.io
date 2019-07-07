---
title: 双重检查加锁错在哪里？
date: 2019-03-24 20:48:33
tags:
  - 单例模式
categories: Java
---

双重检查加锁是实现（多线程版本）单例模式的典型错误：

```Java
public class SingletonError {
    private static SingletonError instance;

    public static SingletonError getInstance() {
        if (instance == null) {
            synchronized (SingletonError.class) {
                if (instance == null) {
                    instance = new SingletonError();
                }
            }
        }

        return instance;
    }
}
```

>其实把 `synchronized` 加在 `getInstance` 方法上就没有问题了，不过在早期的 JVM 上，`synchronized` 很慢，所以程序员们想方设法 *避免同步*。

<!-- more -->

## 问题来源

回想并发问题的三大来源：

* 缓存导致的 **可见性** 问题
* 进程切换导致的 **原子性** 问题
* 编译优化导致的 **有序性** 问题

以上实现存在 **有序性** 和 **原子性** 两方面的问题。

出问题的代码是：

```Java
instance = new SingletonError();
```

我们猜想对象初始化的执行顺序是：

1. 分配堆内存 M；
1. 在堆内存 M 上初始化创建 `Singletonerror` 对象；
1. 将 M 的地址赋值给 `instance` 变量；

实际上，经过编译器或缓存优化后，可能发生指令 **重排序**，2、3 两步的顺序被颠倒：

1. 分配堆内存 M；
1. 将 M 的地址赋值给 `instance` 变量；
1. 在堆内存 M 上初始化创建 `Singletonerror` 对象；

这在 **单线程** 中没有任何影响，因为无论如何，退出同步块之后，1、2、3 三步肯定执行完毕，初始化必然正常完成。但在 **多线程** 中，若其他线程能获取 *将 M 的地址赋值给 `instance` 变量* 后的 `instance` 变量，因为此时 `instance` **还未初始化**，因此可能出现错误。

以上不是危言耸听，考虑如下执行顺序：

1. 线程 A 调用 `getInstance`，并成功进入同步块执行初始化：
  * 分配内存 M；
  * 将 M 的地址赋值给 instance 变量；
1. 线程 A 还没来得及在内存 M 上 **创建 `SingletonError` 对象**，就用完了时间片，无法继续执行，此时切换线程 B 运行；
1. 线程 B 调用 `getInstance`，它发现 `instance != null`，所以直接返回 `instance`，但该对象 **并未初始化**；

>第三步中线程 B 调用 `getInstance` 时，为什么能看到 `instance != null` 呢？
>
>答案是无法保证，既可能看到，也可能看不到：
>* 若看到 `instance == null`，此时尝试进入同步块，线程 A 退出同步块后确保已经完成初始化，此时线程 B 再次进入同步块，发现 `instance != null`，直接返回，此时没有任何问题；
>* 若看到 `instance != null`，则直接返回 `instance`，此时可能返回未初始化的对象，也可能返回初始化完成的对象；

## 如何修复

解决办法有很多，例如：

* 将 `instance` 声明为 `volatile` 变量；
* 将 `getInstance` 声明 `synchronized` 方法；

第二种办法导致 **同一时间** 只有一个线程能调用 `getInstance`，虽然能解决问题，但效率较低。

因此，一般采用第一种办法，其原理为：

* `volatile` 变量规则：对 `volatile` 变量的 **写操作** happens-before **读操作**；
* 禁止 `volatile` 变量的 **写操作** 与之前的内存操作（无论读写）重排序；

>Java 1.5 之前，即使加 `volatile` 也无法修复双重检查加锁的 bug！Java 1.5 新增 `volatile` 变量 happens-before 规则后，才得以修复。

考虑对象初始化的 3 个操作：

1. 分配堆内存 M；
1. 在堆内存 M 上初始化创建 `Singletonerror` 对象；
1. 将 M 的地址赋值给 `instance` 变量；

因为 `instance` 是 `volatile` 变量，且第三步是对 `instance` 的写操作，所以 **禁止** 第三步与前面的任何操作重排序，所以不会发生前面所说的重排序问题。

又因为对 `volatile` 变量的 **写操作** happens-before **读操作**，所以线程 A 执行完初始化第三步后，线程 B 可以立即“看到” `instance` 的新值。

在这两重保证下，双重检查才能正确创建单例对象。

## 停止使用

最后，虽然双重检查加锁的 bug 可以通过多种方法修复，但 **永远不要使用** 该方法实现单例！

推荐两种更好的方式：

* 静态内部类
* 枚举

---

参考：

* [Does the new memory model fix the "double-checked locking" problem?](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#dcl)
* [The "Double-Checked Locking is Broken" Declaration](https://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)
* [为什么说java双重检查(double-check)可能会失效？](https://www.zhihu.com/question/65278232)
