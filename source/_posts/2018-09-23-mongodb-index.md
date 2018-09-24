---
title: MongoDB | 索引
date: 2018-09-23 23:17:10
tags:
  - MongoDB
  - 索引
categories: 技术
---

## 基本术语

[MongoDB 术语表](https://docs.mongodb.com/manual/reference/glossary/)

### index/key/data page

MongoDB 索引由 key 和 data page 两部分组成：

* key 是文档的 **字段集合**；
* data page 是存储在 disk 上的 document 的入口，有时也叫 pointer；

数据库中的索引（index）非常像书本的“索引”，作用是加速读操作，key 类似书本索引的关键字，而 data page 类似页码。

### covered query/fetch

一般而言，读操作分为两步：

1. 由索引的 key 找到目标文档的 data page；
2. 根据 data page 将文档从磁盘 fetch 出来；

若 **所需字段** 都在索引中，则不需要磁盘 fetch 就能获取所需数据，这就是 covered query。

查询覆盖（covered query）一般用于高并发、性能要求高的场景，可进一步加速读操作，但随之而来的是索引变大，需要更大的内存支撑。

### ixscan/collscan

ixscan（索引扫描）和 collscan（集合扫描）效率差别很大：

* 未命中索引时为 collscan，此时需要 fetch 磁盘上的所有文档才能完成查询，时间复杂度为 O(n)，查询效率随文档数量增加而急剧下降；
* 命中索引时为 ixscan，由于索引的底层数据结构（B 树）具备 O(logn) 的查询效率，因此即使文档数量急剧增加，查询效率下降不明显；

<img src="/images/mongodb/ixscan-collscan.png" alt="ixscan vs collscan" style="width: 400px;"/>

### query shape

[query shape](https://docs.mongodb.com/manual/reference/glossary/#term-query-shape) 决定 [query plan](https://docs.mongodb.com/manual/core/query-plans/)。

例如 `db.test.find({name: "Mike", age: 10})` 和 `db.test.find({name: "Bob", age: 20})` 的 query shape 相同，都是 `{name: 1, age: 1}`，因此这两个查询的 query plan 也相同，将使用相同的 index 加速查询。

### index prefix

复合索引有索引前缀的概念，例如：

```
db.test.createIndex({firstName: 1, lastName: 1, gender: 1, age: 1})
```

以上索引有 3 个前缀：

```
{firstName: 1}
{firstName: 1, lastName: 1}
{firstName: 1, lastName: 1, age: 1}
```

以上 3 个索引都是 `{firstName: 1, lastName: 1, gender: 1, age: 1}` 的前缀，将被其覆盖，没有必要单独创建，以避免浪费内存、磁盘空间。

### selectivity

过滤性是指 index 过滤数据的能力，创建索引时，过滤性是重要的参考依据。

假设集合中有 10000 条文档，且：

* 满足 `a = 100` 的文档有 1000 条
* 满足 `b = 10` 的文档有 100 条
* 满足 `c = 1` 的文档有 10 条

则就过滤性而言，c > b > a，按照 c 过滤后的结果集最小。

若查询条件为 `a == 100 && b == 10 && c == 1`，且只能创建一个索引，则选择 c。

### 复合索引 vs 多个单独索引

接着前面的例子，对于 `a == 100 && b == 10 && c == 1`，考虑两种索引方式：

* 复合索引 `{a: 1, b: 1, c: 1}`
* 3 个单独索引：
  * `{a: 1}`
  * `{b: 1}`
  * `{c: 1}`

如果使用 3 个单独索引，MongoDB 需要先分别为每个索引查询，得到 3 个结果集，然后在内存中取 3 个结果集的交集，非常耗时；如果是复合索引，则只会得到 1 个结果集，没有取交集的动作。

3 个单独索引还不如仅创建 `{c: 1}` 一个索引，如果你发现自己正在使用多个单独索引，那就要仔细考虑下了，一般会有更好的索引方式。

在传统的关系型数据库中，多个单独索引比在 MongoDB 中常见一些，原因是 MongoDB 是分布式数据库，为每个索引单独查询，然后取交集的代价非常大，因此 MongoDB 很不推荐这种做法。

## 索引原理

MongoDB 有以下索引类型：

* 单字段索引（single-field index）
* 复合索引（compound index）
* 多键索引（multikey index）
* 地理位置索引（geospatial index）
* 全文索引（text index）

其中单字段索引可以认为是只有一个字段的复合索引，而多键索引是建立在数组上的索引。

### 索引的数据结构

不管是 NoSQL 还是关系型数据库，索引的原理基本都是相同的，其数据结构为 B/B+ 树，注意 B 树并不是 binary tree，而是 self-balancing tree。

索引可以用数组实现，也可以用 B 树实现，两者都可以实现高效查找：

* O(logn) 查找时间复杂度

区别是插入时，维护数组的有序性代价非常高，而 B 树代价相对较低，因此数据库一般用 B 树实现数组。

虽然实际是用 B 树实现的，但可以用数组来讲解。

#### 单字段索引

索引 `{a: 1}`：

```
db.test.insert({a: 1})    [1]
db.test.insert({a: 10})   [1, 10]
db.test.insert({a: 5})    [1, 5, 10]
db.test.insert({a: 7})    [1, 5, 7, 10]
db.test.insert({a: 3})    [1, 3, 5, 7, 10]
```

当插入、删除元素时，索引总是 **保持有序**，因为在有序 B 树/数组上，可以采用注入二分查找等算法，可以达到 O(logn) 的时间复杂度。

#### 复合索引

索引 `{a: 1, b: 1}`：

```
db.test.insert({a: 1, b: 1})    [{a: 1, b: {1}}]
db.test.insert({a: 1, b: 10})   [{a: 1, b: {1, 10}}]
db.test.insert({a: 5, b: 10})   [{a: 1, b: {1, 10}}, {a: 5, b: {10}}]
db.test.insert({a: 5, b: 7})    [{a: 1, b: {1, 10}}, {a: 5, b: {7, 10}}]
```

对于 `db.test.find({a: 5, b: 7})`，首先在 `[1, 5]` 上搜索 `a == 5`，然后在 `[7, 10]` 数组上搜索 `b == 7`，因为是有序数组，可以用二分查找，时间复杂度为 O(logn)。

### 索引作用：过滤

索引最初的目的：加速查询。

假设索引为 `{a: 1, b: 1, c: 1}`，且索引内容为：

<img src="/images/mongodb/index-filter-1.png" style="width: 400px;"/>

精确查询 `find({a: 2, b: 2, c: 3})` 过程如下：

<img src="/images/mongodb/index-filter-2.png" style="width: 400px;"/>

范围查询 `find({a: 2, b: {$gte: 2, $lte: 3}, c: 1})` 过程如下：

<img src="/images/mongodb/index-filter-3.png" style="width: 400px;"/>

范围查询 `find({a: 2, b: 3, c: {$gte: 2, $lte: 4}})` 过程如下：

<img src="/images/mongodb/index-filter-4.png" style="width: 400px;"/>

无论是精确查询，还是范围查询，利用索引都能达到很好的查询效率。

### 索引作用：排序

索引是有序的，如果按照索引顺序进行 **遍历**，则遍历结果也是有序的，可省去对结果集进行 **内存排序**，因此索引有助于排序。

>注意：
>* 并非是先查找到一个结果集，然后用索引对该结果集进行排序；
>* 索引只有助于部分排序，内存排序有时无法避免；

假设有索引 `{a: 1, b: 1, c: 1}`，对于 `find({a: 2}).sort({b: -1, c: 1})`，借助索引，可以避免内存排序：

<img src="/images/mongodb/index-sort-1.png" style="width: 400px;"/>

### 索引顺序

索引顺序可能影响查询和排序。

#### 查询

查询：`find({a: 2, b: {$gte: 2, $lte: 3}, c: 1})`。

若索引为 `{a: 1, b: 1, c: 1}`，则查询过程如下：

<img src="/images/mongodb/index-order-filter-1.png" style="width: 400px;"/>

* 需要 4 次二分查找；

若索引为 `{a: 1, c: 1, b: 1}`，则查询过程如下：

<img src="/images/mongodb/index-order-filter-2.png" style="width: 400px;"/>

* 需要 3 次二分查找；

范围查找的键最好在索引的最后，以避免过早分叉，一分叉，后面的所有查询操作数量要翻倍。

#### 排序

索引顺序将影响索引是否能帮助排序。

假设查询为：`find({a: 2, b: {$gte: 2, $lte: 3}}).sort({c: 1})`。

若索引顺序为 `{a: 1, b: 1, c: 1}`，则执行过程如下：

<img src="/images/mongodb/index-order-sort-1.png" style="width: 400px;"/>

该索引无法帮助排序，因为按 b 过滤之后出现分叉，每个 c 分支内是有序的，但无法保证两个 c 分支之间的顺序，因此必须 **内存排序**。

若索引顺序为 `{a: 1, c: 1, b: 1}`，则执行过程如下：

<img src="/images/mongodb/index-order-sort-2.png" style="width: 400px;"/>

此时在第二层可以按 c 的索引顺序进行遍历，对于每个 c 元素，再继续进行范围查找，最后的结果是 c 升序的，因此无需内存排序。

## 创建索引

创建索引是一项非常耗时的操作，因为需要读取磁盘上的所有文档。

MongoDB 提供两种创建方式，通过 `background` 字段进行控制：

* 前台创建
* 后台创建

其中前台方式会锁住集合，造成暂时的服务不可用，因此几乎不推荐使用；后台方式不锁集合，但创建时间要长一些。

考虑索引顺序可能影响过滤、排序，推荐按如下方式创建索引：

1. 精确匹配字段：最前面
2. 排序字段：中间
  1. 最大化利用索引排序，避免内存排序
3. 范围匹配字段：最后面
  1. 减少过滤时的比较次数，提升过滤效率

按照该原则创建出的索引一般是最优的。

## 使用原则

**1. 保证内存足以容纳索引（最基本）**

>使用 `db.stats().indexSize` 查看索引大小，单位为字节。

CPU 缓存、内存、硬盘之间速度差别：

* 缓存：馒头已经在嘴边，只需要咬一口；
* 内存：满足在盘子里，需要拿起来再咬一口；
* 机械硬盘：需要播种小麦、收割、磨面、做馒头，最后才能吃；
* SSD：先去超时买面粉，然后做馒头，最后吃；

若内存太小无法容纳索引，MongoDB 将使用 LRU 算法将索引在内存和硬盘之间进行交换。

因此若某查询所需索引不在内存，MongoDB 需要先将硬盘上的索引加载到内存中，然后再借助索引进行查找，最后的性能非常差，通常是无法接受的：

<img src="/images/mongodb/mongodb-memory-model.png" style="width: 400px;"/>

**2. 一个查询至少要命中一个索引**

MongoDB 认为耗时超过 100ms 的查询为慢查询，并将其记录在日志中。

**3. 一个索引至少要被一个查询命中**

否则可以直接删除，因此索引是有代价的：

* 影响写入操作
* 占用内存空间

使用 `db.test.aggregrate([{$indexStats: {}}])` 可查看上次 MongoDB 重启后的索引命中情况。

**4. 在保证查询效率的前提下，索引应尽量少**

## 特殊索引

### 哈希索引

索引通过比较索引键进行工作，索引键越长，则比较操作消耗的 CPU 资源越多，而且当键大于 1024 字节时，MongoDB 禁止文档插入。

但如果需要索引的键就是非常长（文件路径、URL 路径等）怎么办呢？

此时用哈希索引，可以显著减少键的长度。

### 部分索引

有时仅需对部分文档创建索引，此时可以用部分索引，可减少索引大小，节省内存使用。

典型应用场景为：

* 状态机
* 消息队列

例如，若仅对 `init` 和 `error` 两个状态的文档创建索引，则：

```
db.test.createIndex({
  stat: 1
}, {
  partialFilterExpression: {
    stat: {
      $in: ['init', 'error']
    }
  }
})
```

### TTL 索引

TTL 索引可自动清理不重要的过期数据，例如过期的 session。

```
db.test.createIndex({
  created: 1
}, {
  expireAfterSeconds: 3600 * 24
})
```

TTL 索引过期后，将自动删除对应的文档。

### 全文索引

关系型数据库的索引，无法支持 **模糊匹配**（`like`），MongoDB 索引也一样，无法支持 **正则表达式**。

唯一的例外是前缀固定的正则表达式，这种是可以命中索引的，但前缀固定的应用场景太局限了：

```
{a: 'This is a tree.'}

find({a: /tree/})    // 模糊匹配不能命中索引
find({a: /^tree/})   // 前缀固定的模糊匹配可以命中索引
```

第二条虽然命中了索引，但并没有什么卵用，此时可以用全文索引取代正则表达式。

>实际上如果真要用全文索引，还不如直接上 ES。

## 常见问题

#### 1. 命中索引的查询是否一定更快？

一般而言，命中索引查询速度更快。

索引的作用是从一个大数据集中过滤出一个 **小数据集**，而 OLAP 应用（分析型应用）需要处理大量数据（比如需要加载一个月的全部数据），因此 OLAP 场景中命中索引不一定更快，此时可能集合扫描要更快。

#### 2. 有多个可用索引，最后会使用哪个？

若某查询语句可能使用多个索引，则首次执行时 MongoDB 将为所有 index 分别生成查询计划，并执行所有计划，第一个查询出 101 条数据的 index 将被选中，以后执行该查询时，将直接使用该索引。

因此首次执行查询时，速度可能会稍微慢一点，而且 MongoDB 重启后需要重新进行评估，当然也可以通过指令强制进行更新。

#### 3. 查询慢？

* 是否命中索引？
* 是否有类似 `toArray` 的操作，导致将结果集一次性加载到内存中？

#### 4. CPU 占用高？

正常场景中，MongoDB 几乎不怎么消耗 CPU。若未命中索引，CPU 利用明显升高。

#### 5. IO 高（最常见）？

* 未命中索引
  * MongoDB 会把文档从磁盘加载到内存中以进行查询操作，从而造成大量磁盘 IO；
* 内存不足
  * MongoDB 内存将与硬盘进行交换，以便将所需索引从磁盘加载到内存，也会造成大量 IO；
  * `nYields` 字段

其中 3-5 是常见的性能问题，性能问题的原因非常多，这里仅仅从索引角度进行分析，并不全面。
