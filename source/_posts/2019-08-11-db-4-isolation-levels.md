---
title: 例说隔离级别
date: 2019-08-11 08:57:24
tags:
  - 事务
categories: 数据库
---

事务的 ACID 四大属性中，CD 相对独立，而 AI 相互影响，可以生出很多变化。

原子性含义为：组成事务的多个写操作，若中间有失败，则：

1. 整个事务被终止；
2. 已执行的写操作被回退；

原子性解决了 **部分更新** 问题，即事务成功提交 or 回退 **后**，DB 不会 **生成** 部分更新的脏数据，因此更准确的说法是，原子性解决了部分更新的 **生成问题**。

部分更新还有 **可见性问题**：并发执行的事务，是否能看到其他事务的中间执行结果？

<!-- more -->

可见性是并发问题的一种，并发执行的事务还会遇到 **竞态条件** 等其他并发问题。

并发问题由隔离性（Isolation）解决，不同隔离级别，在不同程度上解决了事务之间的并发问题，并引入了不同程度的性能惩罚。

事务的典型并发问题有 3 个：

1. 脏读
2. 不可重复读
3. 幻读

四种隔离级别分别对应：

1. read uncommitted
2. read committed：解决脏读
3. repeatable read：解决脏读、不可重复读
4. serializability：解决脏读、不可重复读、幻读

## read uncommitted

事务可以读取其他事务 **未提交** 的写操作，即脏读。

因为其他事务还未提交，所以脏读有以下问题：

1. 脏读结果可能违反一致性（ACID 中的 C）；
2. 脏读结果未持久化，其他事务若终止，则脏读数据在 DB 中丢失；

因此 read uncommitted 是最不可靠的隔离级别，它没有解决任何事务并发问题。

### 示例

>环境：MySQL + InnoDB。

假设有 `counter` 表：

```
mysql> desc counter;
+-------+---------+------+-----+---------+----------------+
| Field | Type    | Null | Key | Default | Extra          |
+-------+---------+------+-----+---------+----------------+
| id    | int(11) | NO   | PRI | NULL    | auto_increment |
| c     | int(11) | YES  |     | NULL    |                |
+-------+---------+------+-----+---------+----------------+
```

开启两个会话 t1、t2，分别设置会话的隔离级别为 read uncommitted：

```
mysql> set session transaction isolation level read uncommitted;
```

`counter` 表中已经有一条数据：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    0 |
+----+------+
```

第一步，在 t1 中手动开启一个事务，并更新该条数据 `c = 2`：

```
mysql> start transaction;
mysql> update counter set c = 2 where id = 1;
```

在 t2 查看，数据已经被修改：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    2 |
+----+------+
```

此时 t1 事务还未提交，t2 就看到了其更新结果，的确是 read uncommitted 的行为。

第二步，若 t1 事务提交成功还好，但若 t1 事务回滚：

```
mysql> rollback;
```

t2 再次查看，发现 `c` 又变回 0：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    0 |
+----+------+
```

从事务的持久性（ACID 中的 D）角度看，`c` 的值从来没有变化过，但会话 t2 却看到它变化了，这就是脏读。

## read committed

事务只能读取其他事务已经提交的写操作，从而避免了脏读问题。

read committed 依然存在问题：

>两个并发事务 t1、t2，t1 修改了数据，t2 在 t1 **提交前后**，分别读取数据，则前后两次读取结果不同，即 **不可重复读**。

在事务 t2 内部，两次读取结果不同，肯定有问题，因为 Isolation 的目标结果是并发执行的事务 **互不影响**，不可重复读说明还是有影响。

### 示例

t1 会话的隔离级别对实验没有影响，因为它是写的一方，将 t2 隔离级别设置为 read committed，同时开启一个事务：

```
mysql> set session transaction isolation level read committed;
mysql> start transaction;
```

第一步，t1 开启事务，并执行更新：

```
mysql> start transaction;
mysql> update counter set c = 2 where id = 1;
```

此时 t1 事务未提交，t2 看不到更新结果，无 **脏读**：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    0 |
+----+------+
1 row in set (0.01 sec)
```

第二步，t1 提交事务：

```
mysql> commit;
```

注意此时 t2 事务未提交，还在执行中，**再次** 查询，发现 `c` 发生变化：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    2 |
+----+------+
1 row in set (0.01 sec)
```

t2 事务的执行过程，在 t1 事务 **提交前后**，同一条语句重复查询的结果不同，这就是不可重复读。

## repeatable read

同一事务执行过程中，脏读、不可重复读都不够理想，相比之下，重复查询结果保持一致要更好一些，这就是 repeatable read。

如何实现可重复读呢？

一种方式是：

1. **事务开始时**，获取整个表的 **快照**；
2. 各个事务执行过程中，查询、更新都在各自快照上执行；
3. 事务提交时，将改动 **回写** 到真正的数据； 

自然不会出现脏读和不可重复读，但有如下问题：

1. 事务提交时，各个事务将自己更新回写到真实数据，可能发生：
   1. 写入冲突，如主键冲突导致只有一个事务提交成功；
   2. 更新覆盖，多个事务更新同一份数据；
2. 事务中的查询操作仅作用于快照，可能发生：
   1. 事务 t1 更新满足条件 c 的记录；
   2. 事务 t2 新增满足条件 c 的记录；
   3. t1 t2 提交后，DB 存在满足条件 c 但又没更新的记录；

如果 DB 真实数据果真与快照一致，则不可能出现以上问题，从事务自己的角度看，就像 DB 中突然冒出一些幽灵数据一样，因此称之为幻读。

### 示例

将 t2 的隔离级别设置为 repeatable read，同时开启一个事务：

```
mysql> set session transaction isolation level repeatable read;
mysql> start transaction;
```

此时 t2 查询结果为：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    0 |
+----+------+
```

第一步，在 t1 中开启事务，执行更新，并提交事务：

```
mysql> start transaction;
mysql> update counter set c = 2 where id = 1;
mysql> commit;
```

此时 t1 查询时已经更新成功：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    2 |
+----+------+
```

第二步，t2 查询，发现依然是老数据，因为 t2 事务还在执行中，因此重复查询的结果保持不变，实现了可重复读：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    0 |
+----+------+
```

注意，DB 中 `id = 1` 的实际记录已经被 t1 修改，t2 看到的是自己的快照，属于“过期数据”。

第三步，在 t2 中将 `c` 的值 +3：

```
mysql> update counter set c = c + 3 where id = 1;
```

从 t2 事务角度看，`c` 应该更新为 3，但实际是 5：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    5 |
+----+------+
```

这就是幻读，明明是在 0 的基础上 +3，怎么最后结果是 5，见鬼了！

注意，此时 t2 事务还未提交，若 t1 事务的隔离级别是 read uncommitted，则 t1 查询结果也是 5：

```
mysql> set session transaction isolation level read uncommitted;
mysql> start transaction;
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    5 |
+----+------+
```

若 t1 事务隔离级别 read committed，则 t1 查询结果仍然是 2：

```
mysql> set session transaction isolation level read committed;
mysql> start transaction;
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    2 |
+----+------+
```

## serializability

串行化是最严格的隔离级别，可解决脏读、不可重复读、幻读，在该级别下，并发事务是 **串行** 执行的，事务没有自己的快照，它们都在同一份数据上执行。

串行化虽然 **彻底解决** 了事务的并发问题，但随之而来的是巨大的性能开销，因此实际应用非常少。

### 示例

DB 中有两条记录：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    0 |
|  4 |   66 |
+----+------+
```

将 t1、t2 的隔离级别都设置为串行化：

```
mysql> set session transaction isolation level serializable;
mysql> start transaction;
```

第一步，t1 执行更新：

```
mysql> update counter set c = 2 where id = 1;
Query OK, 1 row affected (0.00 sec)
```

第二步，t2 同样执行更新：

```
mysql> update counter set c = c + 3 where id = 1;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

* 该更新会 **阻塞** 一段时间，然后报超时错误。

因为事务是串行化的，且 t1 正在执行对 `id = 1` 记录的更新，所以其他事务对 **该条记录** 的 **读、写** 操作都会阻塞，

注意，t1 事务仅将 `id = 1` 的记录锁住，t2 事务可以操作其他记录：

```
mysql> update counter set c = 100 where id = 4;
Query OK, 1 row affected (0.01 sec)
```

第三步，t1 事务提交：

```
mysql> commit;
Query OK, 0 rows affected (0.01 sec)
```

第四部，t2 事务就可以再次更新 `id = 1` 记录了：

```
mysql> update counter set c = c + 3 where id = 1;
mysql> commit;
```

最后数据为：

```
mysql> select * from counter;
+----+------+
| id | c    |
+----+------+
|  1 |    5 |
|  4 |  100 |
+----+------+
```

可以想象，在并发量很大的场景下，串行化的效率是非常低的。
