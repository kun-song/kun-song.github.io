---
title: try-with-resource 全面解析
date: 2019-07-07 10:23:29
tags:
  - try-with-resource
categories: Java
---

程序中不可避免使用各种资源，如文件句柄（文件、Socket）、IO 流、数据库连接等，资源使用完后，必须及时 **关闭**，否则会发生资源泄漏，资源占用达到操作系统上限后将无法获取新资源，只能重启应用才能解决。

Java 7 之前使用 try-catch-finally 管理资源，Java 7 新增了可以自动管理资源的 try-with-resource，它本质上是个语法糖，但也不仅仅是语法糖，它还解决了 Java 存在已久的异常屏蔽问题。

<!-- more -->

## try-catch-finally 的问题

>以下所有示例，都未在 `writing` 方法内处理异常，异常由其调用者处理。

### 待改进代码

先看一段新手代码：

```Java
private static void writing() throws IOException {
    DataOutputStream out = new DataOutputStream(new FileOutputStream("test"));
    out.writeInt(666);
    out.writeUTF("Hello");
    out.close();
}
```

有 4 个可能抛出的异常：

* `new FileOutputStream` 可能抛出 `FileNotFoundException`；
* `writeInt`、`writeUTF`、`close` 可能抛出 `IOException`；

以上代码涉及的资源是文件描述符（`FileDescriptor`），它是 `FileOutputStream` 的一个状态字段，`DataOutputStream` 通过装饰器模式对 `FileOutputStream` 进行增强，本身不占用资源。

下面分析下这 4 个异常的影响。

第一个异常不会造成资源泄漏，因为抛出 `FileNotFoundException` 时，还没创建 `FileDescriptor` 对象：

```Java
public FileOutputStream(File file, boolean append) throws FileNotFoundException {
    ...
    if (file.isInvalid()) {
        throw new FileNotFoundException("Invalid file path");
    }
    this.fd = new FileDescriptor();
    ...
}
```

第二、三个有问题，若 `writeInt` 和 `writeUTF` 抛出异常，则 `DataOutputStream.close` 无法执行，进而导致 `FileOutputStream.close` 无法执行，`DataOutputStream` 只是一个装饰器，没有占用任何操作系统资源，不关闭也没啥影响，但 `FileOutputStream` 占用了文件描述符，不关闭则发生资源泄漏。

第四个也有问题，若 `close` 抛出异常，问题同上。

### 第一次优化：关闭资源

Java 7 之前使用 try-catch-finally 关闭资源，确保无论是否发生异常，finally 中的资源关闭代码都会执行：

```Java
private static void writing() throws IOException {
    DataOutputStream out = new DataOutputStream(new FileOutputStream("test"));

    try {
        out.writeInt(666);
        out.writeUTF("Hello");
    } finally {
        out.close();
    }
}
```

### 第二次优化：“解决”异常屏蔽

这里有个隐藏的问题：`close` 也可能抛出异常，若 `writeInt` 和 `close` 都抛出异常，大多数语言只允许方法每次抛出 **一个** 异常，Java 也不例外，因此 `writing` 的调用者将只能得到 `writeInt` 或 `close` 的异常。

>该现象被称为“异常屏蔽”。

至于实际是哪个，实验一下：

```Java
public class MayException implements AutoCloseable {

    public void tryHi() throws Exception {
        throw new Exception("tryHi failed");
    }

    @Override
    public void close() throws IOException {
        throw new IOException("close failed!");
    }
}

private void test() throws Exception {
    MayException m = new MayException();
    try {
        m.tryHi();
    } finally {
        m.close();
    }
}
```

`test()` 运行输出：

```Java
Exception in thread "main" java.io.IOException: close failed!
  at MayException.close(MayException.java:11)
  at Main.test(Main.java:35)
  at Main.main(Main.java:7)
```

调用者实际得到的是 finally 中抛出的异常，虽然问题根因是 `m.tryHi()`，但调用者得到的异常却是 `m.close()` 抛出的，这会极大干扰问题定位。

解决方式有两种：

1. 手动忽略不重要的异常；
2. 异常嵌套；

第一种，在 finally 中嵌套 try-catch，手动忽略 `close()` 抛出的异常：

```Java
private static void test() throws Exception {
    MayException m = new MayException();
    try {
        m.tryHi();
    } finally {
        try {
            m.close();
        } catch (Exception e) {
            // ignore
        }
    }
}
```

此时 `test()` 可以抛出正确的异常，但丢失了 `close` 抛出的异常：

```Java
Exception in thread "main" java.lang.Exception: tryHi failed
  at MayException.tryHi(MayException.java:6)
  at Main.test(Main.java:33)
  at Main.main(Main.java:7)
```

第二种，异常嵌套：

```Java
private static void test() throws Exception {
    MayException m = new MayException();
    Throwable t = null;
    try {
        m.tryHi();
    } catch (Exception e) {
        // 只为获取 e 的引用，继续抛出 e！
        t = e;
        throw e;
    } finally {
        if (t != null) {
            try {
                m.close();
            } catch (Exception e) {
                // 嵌套异常
                t.initCause(e);
            }
        } else {
            m.close();
        }
    }
}
```

此时 `test()` 没有丢失异常信息，但异常嵌套只应该用于存在 **因果关系** 的异常，这俩实际没有任何关系，同样会误导问题定位：

```Java
Exception in thread "main" java.lang.Exception: tryHi failed
	at MayException.tryHi(MayException.java:6)
	at Main.test(Main.java:21)
	at Main.main(Main.java:7)
Caused by: java.io.IOException: close failed!
	at MayException.close(MayException.java:11)
	at Main.test(Main.java:28)
	... 1 more
```

因此，无论是手动忽略异常，还是手动嵌套异常，都不是完美的解决方式。

### 总结

从上面的例子可以看到，用 try-catch-finally 正确地关闭资源有多么困难：

1. 异常可能被吞 or 错误嵌套，影响问题定位；
2. 编写正确的资源关闭代码没那么容易，资源太多时可能需要大量的嵌套 try-catch-finally；
3. 代码繁琐、冗长，不易读；
4. 可能忘记关闭资源；

## try-with-resource 拯救大家

资源关闭都是些样板代码，手写繁琐又容易出错，因此 Java 7 新增了 try-with-resource 语法糖解放大家的双手，`try()` 块中打开的资源，无论什么情况，都可以确保关闭：

```Java
private static void writing() {
    try (DataOutputStream out = new DataOutputStream(new FileOutputStream("test"))) {
        out.writeInt(666);
        out.writeUTF("Hello");
    }
}
```

并且可保证调用者得到正确的异常信息：

```Java
private static void test() throws Exception {
    try (MayException m = new MayException()) {
        m.tryHi();
    }
}
```

`test()` 输出如下：

```Java
Exception in thread "main" java.lang.Exception: tryHi failed
	at MayException.tryHi(MayException.java:6)
	at Main.test(Main.java:19)
	at Main.main(Main.java:7)
	Suppressed: java.io.IOException: close failed!
		at MayException.close(MayException.java:11)
		at Main.test(Main.java:18)
		... 1 more
```

不仅能得到 `tryHi` 抛出的异常，`close` 抛出的异常也没有丢失，而且也不是嵌套形式，比前面两种方式都好。

## `AutoCloseable` 接口

然而并非所有资源都可以被 try-with-resource 自动管理，必须实现 `AutoCloseable` 接口才行，try-with-resource 将自动调用 `AutoCloseable.close` 方法：

```Java
public interface AutoCloseable {
    void close() throws Exception;
}
```

>虽然上面的 `close` 方法声明抛出 `Exception`，但强烈建议实现类声明更 **具体** 的异常，若根本不抛出异常，则建议 **去掉** 异常抛出声明。

值得注意的是，“老旧”的 `Closeable` 接口继承了 `AutoCloseable`：

```Java
public interface Closeable extends AutoCloseable {
    public void close() throws IOException;
}
```

如此一来，现存的已经实现 `Closeable` 的资源类可以直接用在 try-with-resource 中，无需做任何修改。

## 实现原理

### 语法糖

前面说过，try-with-resource 其实是语法糖，反编译看下：

```Java
private static void test() throws Exception {
    MayException m = new MayException();
    // 一个方法只能抛出一个异常，通过 var1 修改 try{} 和 finally{} 两处抛出的异常
    Throwable var1 = null;

    try {
        m.tryHi();
    } catch (Throwable var10) {
        // var10 是实际抛出的第一个异常，使 var1 引用 var10
        // 以便 finally 里修改 var10 的内容
        var1 = var10;
        throw var10;  // 再次抛出
    } finally {
        if (m != null) {
            if (var1 != null) {  // m.tryHi 抛出异常
                try {
                    m.close();
                } catch (Throwable var9) {
                    // 附加异常
                    var1.addSuppressed(var9);
                }
            } else {
                // m.tryHi 未抛出异常，则直接调用 close，若 close 抛出异常
                // 则该异常为 test 最终抛出的异常
                m.close();
            }
        }
    }
}
```

>注意，不同版本的 Java 编译器的 try-with-resource 实现细节不同，以上是 Java 8 编译的结果。

不难看出，编译器自动生成的代码与手写代码几乎相同，只有两点不同：

1. 增加 `if (m != null)` 判断，手写代码时，我们很清楚 `m` 不可能为空，但编译器为保险起见，还是加了这个判断；
2. 将 `initCause` 替换为 `addSuppressed`；

### 解决异常屏蔽

`Throwable.addSuppressed` 是 Java 7 新增的专门解决异常屏蔽的方法，前面说过 `initCause` 用于嵌套两个存在因果关系的异常，而 `addSuppressed` 用于连接两个 **独立的、没有任何关系的** 异常，最常见的场景就是 try{} 和 finally{} 同时抛出异常。

## 避免踩坑

并非用了 try-with-resource 就一定万事大吉，原因很简单，因为完全可以实现一个空 or 不关闭资源的 `close`，当然这是极端的做法，但实现不合理 or 不正确 or 特别的 `close` 却极有可能存在。

### 错误的链式声明

```Java
private static void compress(String input, String output) throws IOException {
    try (
        FileInputStream fis = new FileInputStream(input);
        GZIPOutputStream gos = new GZIPOutputStream(new FileOutputStream(output))
    ) {
        byte[] buffer = new byte[4096];
        int nread = 0;
        while ((nread = fis.read(buffer)) != -1) {
            gos.write(buffer, 0, nread);
        }
    }
}
```

我们知道，编译器将生成代码自动调用 `GZIPOutputStream.close` 和 `FileInputStream.close`，若两个调用都成功那自然没问题。

但 `GZIPOutputStream.close` 有个小小的问题，即它可能抛出异常，且资源关闭在可能抛出异常的代码之后：

```Java
/**
 * Writes remaining compressed data to the output stream and closes the
 * underlying stream.
 * @exception IOException if an I/O error has occurred
 */
public void close() throws IOException {
    if (!closed) {
        finish();  // 可能抛出异常
        if (usesDefaultDeflater)
            def.end();
        out.close();  // 资源关闭
        closed = true;
    }
}
```

`close` 方法首先调用 `finish` 将剩余的数据写入流，然后再关闭流（即其修饰的 `FileOutputStream`），若 `finish` 抛出异常，则 `FileOutputStream` 无法关闭，从而发生资源泄漏。

### 正确的链式声明

作为对比，重新贴下前面 `DataOutputStream` 的例子：

```Java
private static void writing() {
    try (DataOutputStream out = new DataOutputStream(new FileOutputStream("test"))) {
        out.writeInt(666);
        out.writeUTF("Hello");
    }
}
```

这里没有单独声明 `FileOutputStream`，因为 `DataOutputStream.close` 可以正确关闭其底层的 `FileOutputStream`：

```Java
public void close() throws IOException {
    try (OutputStream ostream = out) {
        flush();
    }
}
```

`close` 先执行 `flush`，无论是否成功，最后调用 `ostream.close`，这实际执行的是 `FileOutputStream.close`，因此可以保证关闭 `FileOutputStream`。

### 最佳实践

因此，try-with-resource 能否成功关闭资源，取决于其 `close` 方法的实现细节，若：

1. `close` 可能抛出异常，且
2. 释放资源代码在可能抛出异常的代码之后

则 try-with-resource 无法保证资源正确关闭。

退一步讲，假设 **直接持有** 系统资源的类的 `close` 都能正确释放资源，**间接持有** 资源的类的 `close` 不确定能否正确释放资源，则正确使用 try-with-resource 的原则是：

>**为每个直接持有资源的类单独进行声明，以确保总能调用其 `close` 方法释放资源。**

---

参考：

* [Better Resource Management with Java SE 7: Beyond Syntactic Sugar](https://www.oracle.com/technetwork/articles/java/trywithresources-401775.html)

