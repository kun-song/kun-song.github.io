---
title: ConcurrentModificationException 介绍
date: 2019-11-27 00:21:38
tags:
categories: Java
---

`ConcurrentModificationException` 的名字有点蛋疼，可能让人误解只有 **并发场景** 才会抛出该异常。

<!-- more -->

这里的“并发”实际与 **线程数量** 没有关系，它是指 **遍历集合** 和 **结构化修改集合** 两个动作不能同时发生，更具体指：结构化修改动作不能 **穿插** 在集合遍历动作之间。

## 抛出场景

该异常在 **遍历集合** 时抛出，若遍历集合时 **检测到** 有结构化修改，则抛出，结构化修改包含：

* 添加元素
* 删除元素
* 清空集合

注意，修改已有元素的值并非结构化修改。

### 单线程

#### 问题场景

单线程 **只有一种** 场景会抛出该异常：遍历集合时，同时对集合做 **结构化修改**：

```Java
List<Integer> xs = Stream.of(1, 2, 3, 4, 5).collect(toList());

for (int x : xs) {
    if (x % 2 == 0) xs.remove(x);
}
```

异常：

```
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:909)
	at java.util.ArrayList$Itr.next(ArrayList.java:859)
	at com.satansk.WeekTest.main(WeekTest.java:17)
```

#### 解决方式

无法完全解决，遍历时 **添加** 元素无法避免抛异常，遍历 + 删除元素有两种解决方式：

* `Iterator.remove`
* `Stream.filter`

```Java
List<Integer> xs = Stream.of(1, 2, 3, 4, 5).collect(toList());

Iterator<Integer> it = xs.iterator();
while (it.hasNext()) {
    int x = it.next();
    if (x % 2 ==0) it.remove();
}
```

### 多线程

#### 问题场景

相比单线程，多线程的问题要隐秘很多，因为遍历与结构化修改是 **隐式** 发生的，即：

* 线程 A 遍历集合，同时
* 线程 B 结构化修改集合

同时由于 **可见性** 问题，并非以上两个动作同时发生就 **一定** 会抛出异常，只有当集合 **自己检测到** 并发修改时才会抛出：

```Java
List<Integer> xs = Stream.of(1, 2, 3, 4, 5).collect(toList());

Thread t1 = new Thread(() -> {
    for (int x : xs) System.out.println(x);
});

Thread t2 = new Thread(() -> xs.remove(3));

t1.start();
t2.start();

t1.join();
t2.join();

System.out.println(xs);
```

有时不抛异常：

```
[1, 2, 3, 5]
```

有时抛异常：

```
1
Exception in thread "Thread-0" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:909)
	at java.util.ArrayList$Itr.next(ArrayList.java:859)
	at com.satansk.WeekTest.lambda$main$0(WeekTest.java:19)
	at java.lang.Thread.run(Thread.java:748)
[1, 2, 3, 5]
```

#### 解决方式

可完全解决，使用同步 or 并发安全的集合。

## fail-fast 迭代器

Java 集合中很多迭代器是 fail-fast 迭代器，fail-fast 机制最初是为了避免对集合的 **并发修改**，并发修改通常意味着 **并发错误**，因此当迭代器检测到并发修改时，并不会冒风险继续执行，而是立即失败（fail-fast），因为失败比错误更好。

当检测到并发修改时一定抛出 `ConcurrentModificationException`，但没有抛出该异常并不意味没有发生错误，因为 `modCount` **并非线程安全**，多线程修改时 `modCount` 不一定对其他线程可见，因此抛出该异常的行为是 best-effort 的，即迭代器会努力检测各种错误的发生，但 **不保证** 一定能检测到所有错误。

因此 `ConcurrentModificationException` 只能用来 **检测错误**（detect bugs），即发现该异常说明有 bug，业务逻辑决不能依赖该异常。

---

参考：

* [Javadoc ConcurrentModificationException](https://docs.oracle.com/javase/7/docs/api/java/util/ConcurrentModificationException.html)
