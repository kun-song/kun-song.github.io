---
title: tut -- 简单的 Scala 文档检查工具
date: 2018-11-21 09:00:39
tags:
  - tut
categories: 编程工具
---

我们写技术文档时常常需要贴代码片段，但这类代码的正确性却很难保证，可能在 IDE 中跑的好好的，但粘贴时缺胳膊少腿，最后对读者造成一定的阅读障碍。

Scala 是一门静态强类型语言，编译时即可发现很多问题，深受老夫喜爱，而 tut 则是 Scala 平台解决上述问题的小工具。

tut 使用非常广泛，很多知名 Scala 项目用它编写文档，例如 Cats、Akka 等，其原理非常简单，读取 markdown 文件中用 `tut` 修饰的代码片段，编译执行，将结果作为注释插入代码片段中，通过这种方式，可以保证文档中所有代码都可以正常运行，完美解决开篇的问题。

<!-- more -->

使用前的准备工作：

1. 在 `project/plugins.sbt` 中添加 tut 插件：

  ```Scala
  addSbtPlugin("org.tpolecat" % "tut-plugin" % "0.6.9")
  ```

2. 在 `build.sbt` 中开启插件：

  ```Scala
  enablePlugins(TutPlugin)
  ```

我们来看一个实际使用的例子，Cats 文档中有这么一段示例代码：

```Scala
```tut
val numberOption: Option[Int] = None
val numberFOption: List[Option[Int]] = List(None, Some(2), None, Some(5))

val number = IorT.fromOption[List](numberOption, "Not defined")
val numberF = IorT.fromOptionF(numberFOption, "Not defined")
```.
```

由于社区现在倾向于用 `Chain` 替代 `List`，因此我想修改以上代码，以反映最新的最佳实践，按照直觉，直接替换 `List` 即可：

```Scala
```tut
val numberOption: Option[Int] = None
val numberFOption: Chain[Option[Int]] = Chain(None, Some(2), None, Some(5))

val number = IorT.fromOption[Chain](numberOption, "Not defined")
val numberF = IorT.fromOptionF(numberFOption, "Not defined")
```.
```

修改后进入 sbt 的交互模式执行 `tut` 指令，结果报错如下：

```Scala
[tut] *** Error reported at /Users/satansk/Projects/scala/cats/docs/src/main/tut/datatypes/iort.md:230
<console>:25: error: not found: type Chain
       val numberFOption: Chain[Option[Int]] = Chain(None, Some(2), None, Some(5))
...
```

>`tut` 指令默认编译所有 markdown 文件，耗时会比较久，若想加快速度，可以用 `tutQuick` 进行增量编译，或者用 `tutOnly <path>` 编译指定路径的文件。

提示无法找到 `Chain` 类型，导入即可：

```Scala
```tut
import cats.data.Chain

val numberOption: Option[Int] = None
val numberFOption: Chain[Option[Int]] = Chain(None, Some(2), None, Some(5))

val number = IorT.fromOption[Chain](numberOption, "Not defined")
val numberF = IorT.fromOptionF(numberFOption, "Not defined")
```.
```

有人会问，为啥 `Iort` 不需要导入呢，在 `iort.md` 开头有一段：

```
---
layout: docs
title:  "IorT"
section: "data"
source: "core/src/main/scala/cats/data/IorT.scala"
scaladoc: "#cats.data.IorT"
---
```

我猜因为是 `Iort` 的文档，所以默认在每个代码片段里都会导入 `Iort`，所以就不需要我们自己导入了。

当然，虽然 tut 很简单，但也不至于只有这么点功能，想了解更多内容可以去它的官网看看，这里就不多写了。

---

参考：

* [tut 官网](http://tpolecat.github.io/tut/)
