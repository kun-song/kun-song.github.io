---
title: Scala | 字符串揭秘
tags:
categories: Scala
abbrlink: 557
date: 2018-05-14 23:24:19
---

## `String` 源码剖析

Scala 中的 `String` 其实仅仅是 `java.lang.String` 的别名，在 `scala.Predef` 中可以找到其定义：

```Scala
type String = java.lang.String
```

明明 Scala 的字符串有很多非常有用的函数啊，难道都是错觉？

<!-- more -->

实际上 `scala.Predef` 中还有一个与 `String` 关系重大的隐式函数：

```Scala
@inline implicit def augmentString(x: String): StringOps = new StringOps(x)
```

该函数将 `String` 值隐式转换为 `StringOps` 值，`StringOps` 源码如下：

```Scala
final class StringOps(override val repr: String) extends AnyVal with StringLike[String] {
  ...
  override def apply(index: Int): Char = repr charAt index
  override def slice(from: Int, until: Int): String = ???
```

除了 `slice` 函数以外，其余函数我们也没怎么用过，还得继续深入一层，看下 `StringLike` 的源码：

```Scala
trait StringLike[+Repr] extends Any with scala.collection.IndexedSeqOptimized[Char, Repr] with Ordered[String] {
self =>

  override def mkString = toString

  override def slice(from: Int, until: Int): e

  def * (n: Int): String = 

  def stripLineEnd: String

  def linesWithSeparators: Iterator[String]

  def capitalize: String

  def stripPrefix(prefix: String)

  def stripSuffix(suffix: String)

  def replaceAllLiterally(literal: String, replacement: String): String

  def stripMargin(marginChar: Char): String

  def stripMargin: String = stripMargin('|')
  
  ...
}
```

卧槽，我们平时常用的函数都在这里啊，原来如此！

其实这是 Scala 的一种常见设计模式，即 Type Class 模式（源于 Haskell），通过 `implicit` 转换函数，在不修改第三方库源码（`java.lang.String`）的条件下，实现对第三方库功能增强，用在 `String` 这里非常适合。

## 常用函数

下面是一些 Scala `String` 的常用函数，非常方便。

### 多行字符串

使用 `"""` 包裹即可实现多行字符串，比 Java 方便太多：

```Scala
val s =
  """
    This is
    a scala
    line
  """
```

输出如下：

```Scala
s: String = 
    This is
    a scala
    line
```

若想要 **去掉** 每行开头的空字符串，可以用 `stripMargin`：

```Scala
val a =
  """
    |This is  // 使用默认分割符 |
    |a scala
    |line
  """.stripMargin
```

输出如下：

```Scala
a: String = 
This is
a scala
line
```

`stripMargin` 函数实现如下：

```Scala
/** For every line in this string:
  *
  *  Strip a leading prefix consisting of blanks or control characters
  *  followed by `|` from the line.
  */
def stripMargin: String = stripMargin('|')
```

因此，不必非要用 `|` 分割符，可以使用自定义的分隔符：

```Scala
val b =
  """
    #This is
    #a scala
    #line
  """.stripMargin('#')
```

输出与使用 `|` 完全相同：

```Scala
b: String = 
This is
a scala
line
```

### 字符串插值

>字符串插值是通过将 `String` 隐式转换为 `StringContext` 实现的，Scala 允许我们自定义字符串插值器。

#### 1. s 插值

在任意字符串前面加 `s` 后，就可以在该字符串中使用 **变量** 和 **表达式** 了：

```Scala
val a = 111
val b = 6
val c = s"a + b = ${a + b}"
```

#### 2. f 插值

在字符串前面加 `f`，以简单格式化字符串：

```Scala
val a = 111.111111
val b = 6
val c = f"a * b = ${a * b}%2.2f"
```

* `%2.2f` 指定格式

#### 3. raw 插值

对字符串中的字符 **不做编码**，其他与 s 插值相同。

若有 s 插值如下：

```Scala
val a = 111
val b = s"b = \n$a"

// 输出
a: Int = 111
b: String = b = 
111
```

若不想对 `\n` 转义，则可用 raw 插值：

```Scala
val a = 111
val b = raw"b = \n$a"

// 输出
a: Int = 111
b: String = b = \n111
```

### strip

去掉字符串前后缀：

```Scala
". Hello .".stripPrefix(".").stripSuffix(".").trim
```
