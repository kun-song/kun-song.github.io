---
title: 捋捋 BufferedInputStream 的设计与实现
date: 2019-07-13 11:28:16
tags:
  - BufferedInputStream
  - 装饰器模式
  - try-with-resource
  - AtomicReferenceFieldUpdater
categories: Java
---

`BufferedInputStream` 算是 Java IO 最常用的类之一了，最近看了下它的源码，发现不少有趣的地方。

<!-- more -->

## 蛋疼的装饰器模式

首先，容我吐槽下，我对 Java IO 类的印象是数量超级多、超级难记，这个锅是装饰器模式的，装饰器模式使用不当，很容易造成类数量爆炸，这明显发生在了 Java IO 类上。

从设计层面上看，IO 类似乎设计的非常精妙，类的接口很简单，类之间通过装饰器实现功能的增强，这些从理论上都没错，但从实践上看，个人认为 IO 类设计的非常烂：

1. 类太多，完全记不住；
2. 类太多，不知道怎么组合才能想要的功能；
3. 最常用的功能组合就那么几个，但无法开箱即用；

`BufferedInputStream` 就是第三点的例子，IO 读写 99.999% 都需要使用缓存，但由于无法开箱即用，每次都要写类似这样的代码：

```Java
BufferedInputStream bis = new BufferedInputStream(new FileInputStream("xx"));
```

完全可以设计为默认提供缓存，同时提供无缓存版的方法，这样大家平时只需关注前者，只有少数场景才需要了解后者。但目前这种设计，用户无差别的面对各种常用 + 非常用功能，重点不突出，增加用户的心智负担。

## `close` 方法实现

`close` 方法有两点需要注意：

1. “正确”实现 `close` 以简化 try-with-resource 的使用；
2. `close` 实现中的并发考虑；

### try-with-resource

使用 `BufferedInputStream` 时可以在 try() 链式声明：

```Java
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream(""))) {
    // do something
} catch (Exception e) { 
    // ignore
}
```

我在 [try-with-resource 全面解析](http://songkun.me/2019/07/07/2019-07-07-java-try-with-resource/) 中说过，能否在 try() 中链式声明主要取决于外层类的 `close` 实现：

```Java
public void close() throws IOException {
    byte[] buffer;
    while ((buffer = buf) != null) {
        if (bufUpdater.compareAndSet(this, buffer, null)) {
            InputStream input = in;
            in = null;
            if (input != null)
                input.close();  // 关闭被装饰的 InputStream
            return;
        }
        // Else retry in case a new buf was CASed in fill()
    }
}
```

可见除 `input.close();` 这行外，其他代码都不可能抛出 `IOException`，这可以保证 `input.close();` 一定执行，而 try-with-resource 保证一定调用 `BufferedInputStream.close()`，进而保证被装饰的 `InputStream` 一定被关闭，所以可以用链式声明。

### 并发

`BufferedInputStream` 几乎所有 public 方法都是同步的（用 `synchronized` 修饰），`close` 是唯二的非同步方法，另一个方法简单返回 `true`，自然不需要同步：

```Java
public boolean markSupported() {
    return true;
}
```

为什么 `close` 不是同步方法呢，原因是线程执行 `close` 时，其他线程可能 **同时** 在执行其他方法，比如 `read` 等，如果 `close` 也是同步方法，则必须等其他线程执行 `read` 结束才能开始 `close`，这一等可能是几秒、几分钟，也可能永远等下去，从而永远无法关闭流，因此 `close` 绝不能是同步方法。

实际上 `close` 通过 CAS 方法实现线程安全的流关闭：

```Java
while ((buffer = buf) != null) {
    // buf 为 null 表示流已经被关闭
    if (bufUpdater.compareAndSet(this, buffer, null)) {
        InputStream input = in;
        in = null;
        if (input != null)
            input.close();
        return;
    }
    // Else retry in case a new buf was CASed in fill()
}
```

`buf` 是内部的缓存字段，可能被其他线程通过 `fill()` **并发修改**，修改方式是 **创建一个新 byte[]，然后使 buf 指向该新数组**，所以首先将 `buf` 保存在 `buffer` 副本中，然后通过 CAS 将 `buf` 字段设置为 `null`，如果设置成功，则关闭底层的 `InputStream`，否则不断自旋重试。

## AtomicReferenceFieldUpdater vs AtomicReference

`BufferedInputStream` 内部的缓冲字段为：

```Java
protected volatile byte buf[];
```

前面说过，`close` 通过 CAS 将 `buf` 设置为 `null`，而 `buf` 并非原子引用，不支持 CAS 操作。

因为 `buf` 是 `volatile` 修饰的引用类型，所以可以借助 `AtomicReferenceFieldUpdater` 为该字段添加 CAS 操作：

```Java
/**
 * Atomic updater to provide compareAndSet for buf. This is
 * necessary because closes can be asynchronous. We use nullness
 * of buf[] as primary indicator that this stream is closed. (The
 * "in" field is also nulled out on close.)
 */
private static final AtomicReferenceFieldUpdater<BufferedInputStream, byte[]> bufUpdater =
    AtomicReferenceFieldUpdater.newUpdater(BufferedInputStream.class,  byte[].class, "buf");
```

* `buf` 为 `null` 则表示流已经被关闭，借助 `bufUpdater` 以 CAS 方式将 `buf` 设置为 `null`；

利用 `bufUpdater`，在 `BufferedInputStream` 内部可以对 `buf` 执行各种 CAS 操作，例如 `close`：

```Java
bufUpdater.compareAndSet(this, buffer, null)
```

我们知道 `AtomicReferenceFieldUpdater` 是通过 **反射** 实现的，性能比 `AtomicReference`（`Unsafe` 实现）要差，为什么不直接用 `AtomicReference<byte[]>` 呢：

```Java
protected AtomicReference<byte[]> buf;
```

首先 `BufferedInputStream` 内部只有两处用到了 CAS，其他大部分都是对 **普通数组** 的操作，如果只用 `AtomicReference<byte[]>`，则大量的数组操作需要先 `buf.get()`，然后才能操作数组，非常繁琐。

其次，如果保留 `buf`，只是将 `bufUpdater` 替换为 `AtomicReference`：

```Java
protected volatile byte buf[];
private static final AtomicReference<byte[]> bufUpdater = ...
```

那么每次 `buf` 引用变动时，都需要维持 `bufUpdater` 和 `buf` 之间的对应关系，而且这种维持必须是原子的，增加了很多工作量和复杂性。

因此 `BufferedInputStream` 选择使用 `AtomicReferenceFieldUpdater` 来实现 `buf` 字段的 CAS 操作，是比较合理的选择。
