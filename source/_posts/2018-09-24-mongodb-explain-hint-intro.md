---
title: MongoDB | explain 与 hint
date: 2018-09-24 23:18:03
tags:
  - MongoDB
  - 索引
  - explain
  - hint
categories: 数据库
---

`explain` 和 `hint` 是最常用的 MongoDB 性能调优工具，本文简单总结下两者的使用。

## `explain`

`explain()` 是 MongoDB 最重要的性能诊断工具之一，借助 `explain` 可以获取很多查询语句的执行信息。

对于任何查询语句，**最后** 都可以添加 `explain` 函数，例如：

<!-- more -->

```
> db.users.find({"age" : 42}).explain()
{
    "cursor":"BtreeCursor age_1_username_1",
    "isMultiKey":false,
    "n":8332,
    "nscannedObjects":8332,
    "nscanned":8332,
    "nscannedObjectsAllPlans":8332,
    "nscannedAllPlans":8332,
    "scanAndOrder":false,
    "indexOnly":false,
    "nYields":0,
    "nChunkSkips":0,
    "millis":91,
    "indexBounds":{
        "age":[
            [
                42,
                42
            ]
        ],
        "username":[
            [
                {
                    "$minElement":1
                },
                {
                    "$maxElement":1
                }
            ]
        ]
    },
    "server":"ubuntu:27017"
}
```

下面介绍其中的关键字段。

### `cursor`

从 `cursor` 字段可以看出该查询是否 **命中索引**：

* `BasicCursor`：未命中
* `BtreeCursor`：命中

且命中索引是 `BtreeCursor` 后面跟随命中的索引名字，例如 `age_1_username_1`。

### `millis`

`millis` 为该查询的执行时间，单位为 ms。

### `n`

`n` 是实际返回的文档数量，即结果集的数量。

### `nscanned` 与 `nscannedObjects`

这两个字段代表为完成该查询，MongoDB 背后做的实际工作量：

* `nscanned`：MongoDB 搜索的 index entry 数量；
* `nscannedObjects`：MongoDB 搜索的文档数量；

### `scanAndOrder`

`scanAndOrder` 表示是否需要内存排序：

* `true` 表明无法借助索引排序，只能对结果集进行 **内存排序**；
* `false` 表明可借助索引排序；

内存排序非常慢，且只能用于小数据集，默认情况下，MongoDB 只能对小于 128M 的数据进行内存排序。

### `indexOnly`

`indexOnly` 表示该查询是否被索引覆盖：

* `true` 表明该查询可以直接在索引中获取结果，无需 fetch 磁盘上的实际文档；

索引覆盖需要满足两个条件：

* 索引中包含想要的字段；
* 查询不需要返回其他字段（通过 `find({}, {_id: 0, username: 1, age: 1})` 指定）

### `nYields`

该查询“让出”（yield）资源，让其他写操作先执行的让出次数。

一般由于该查询无法立刻完成导致，例如内存不足无法容纳全部索引，导致该查询所需索引需要从硬盘加载。

### `indexBounds`

`indexBounds` 代表索引的使用情况，包含被检索的索引的范围：

```
"indexBounds":{
    "age":[
        [
            42,
            42
        ]
    ],
    "username":[
        [
            {
                "$minElement":1
            },
            {
                "$maxElement":1
            }
        ]
    ]
}
```

因为 `db.users.find({"age" : 42})` 中 `age` 为精确匹配，所以其上下界都是 42，且没有使用索引第二个字段，所以 `username` 上下界为最大、最小元素。

## `hint`

`hint` 可强制 MongoDB 使用指定的索引进行查询，例如：

```
db.test.find({'age': 14, username: /.*/}).hint('username': 1, 'age': 1)
```

但 MongoDB 自动选择的索引一般就是最优解，`hint` 一般用来辅助调优，所以 `hint` 后一般跟 `explain`。
