---
title: 源码分析 | HashMap
date: 2018-12-02 20:51:06
tags:
  - HashMap
categories: 源码分析
---

`HashMap` 是最常用的数据结构之一，也是面试的常客，了解 `HashMap` 的实现原理对正确、高效地的使用它是非常重要的，本文将细致分析 Java 8 `HashMap` 实现的所有细节。

<!-- more -->

## 继承关系

首先看下 `HashMap` 的类继承：

```Java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

`HashMap` 继承 `AbstractMap`，它提供了很多 `Map` 接口的默认实现，除 `HashMap` 外，Java 集合框架中还有很多它的子类：

* `WeakHashMap`
* `NavigableSubMap`
* `IdentityHashMap`
* `EnumMap`

...

## 状态字段

`HashMap` 定义的状态字段有：

```Java
transient Node<K,V>[] table;
transient Set<Map.Entry<K,V>> entrySet;
transient int size;
transient int modCount;
int threshold;
final float loadFactor;
```

`table` 是 `Node<K, V>` 数组，它的长度要么是 0，要么是 2 的整数幂，注意 `table` 数组并非 `HashMap` 构造时初始化，而是当第一次使用时初始化。

`entrySet` 是 `HashMap` 的 k-v 映射视图，注意 `HashMap` 还有 key 视图和 value 视图，支持它俩的字段在 `AbstractMap` 中定义。

`size` 为该 `HashMap` 中保存的 k-v 映射数量。

`modCount` 表示该 `HashMap` 被结构化修改的次数，结构化修改分为两类：

1. 改变 `HashMap` 中 k-v 映射数量的操作（增、删）
2. 改变 `HashMap` 内部结构的操作（`rehash`）

该字段的作用是：

```
This field is used to make iterators on Collection-views of the HashMap fail-fast. 
```

`HashMap` 提供 3 种集合视图用于遍历 `HashMap`：

* key 集合
* value 集合
* k-v 集合

遍历它们的时候，若 `HashMap` 发生结构化修改，则抛出 `ConcurrentMidificationException`。

`threshold` 注释如下：

```
The next size value at which to resize (capacity * load factor).
```

每次执行 `resize` 时，都会修改 `table` 数组的大小，而 `threshold` 则是下次调用 `resize` 时 `table` 扩容（or 初始化）的目标长度。

`loadFactor` 非常重要，注释如下：

```
The load factor is a measure of how full the hash table is allowed to
get before its capacity is automatically increased.

When the number of entries in the hash table exceeds the product
of the load factor and the current capacity, the hash table is rehashed. 
```

若 `HashMap` 中保存的 k-v 映射数量（即 `size`）超过 `loadFactor * currentCapacity`，就会执行 `resize`，其中 `HashMap` 的 `capacity` 就是 `table` 数组的大小（即桶数量），当前容量即有数据的桶的个数。

上面注释翻译成代码应该是这样：

```Java
if (size > loadFactor * currentCapacity)
    resize();
```

## 构造函数

`HashMap` 有 3 个常见的构造函数：

```Java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: "  loadFactor);

    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

可以看到，这 3 个构造函数其实就设置了两个字段：

* `loadFactor`
* `threshold`

`loadFactor` 很容易理解，而 `threshold` 是下次 `HashMap` 扩容的目标容量，这个值是通过 **当前容量** 计算出的：

```Java
// Returns a power of two size for the given target capacity.
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

`tableSizeFor` 接受当前容量，然后返回比当前容量大的最小的 2 的整数幂，比如 `tableSizeFor(55)` 返回 64，表示下次 `HashMap` 扩容的目标容量为 64。

因此使用 `HashMap` 时，它的容量必然是 2 的整数幂，即 `table` 长度必然为 2 的整数幂，因此可以用如下方式加速 key 查找：

```Java
(n - 1) & hash(key);
```

`n` 为 `table` 数组大小。

## `put`

```Java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

根据注释，若 `HashMap` 已经存在该 `key`，则：

* 更新该 key 绑定的值；
* 返回该 key 绑定的旧值；

若 `HashMap` 不存在该 key，则：

* 添加 key-value
* 返回 `null`

因此 `put` 返回值为 `null` 可能有两种情况：

* key 不存在
* key 旧值为 `null`

所以不能依赖 `put` 的返回值判断 key 是否已经存在。

`put` 实际工作是委托给 `putVal` 实现的，其签名如下：

```Java
// 返回 key 映射的旧值，若不存在，则返回 null
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) 
```

各参数含义如下：

* `key` 和 `value` 是要插入的 k-v
* `hash` 是 `key` 的哈希值
* `onlyIfAbsent` 是否仅当该 `key` 不存在插入，若为 `false` 则修改已有映射
* `evict` 为 `false`，表明正处于创建模式

`put` 对它调用如下：

```Java
putVal(hash(key), key, value, false, true)
```

`onlyIfAbsent` 为 `false`，所以 `put` 会修改已有的 k-v 映射，`evict` 为 `true`，表明当前并非创建模式，哈希值则通过 `hash` 计算，该函数比较重要，是 `HashMap` 实现高效查找、插入的关键，其实现如下：

```Java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

首先注意 `key` 可为 `null`，此时哈希值为 0，因此 `HashMap` 可以允许 key 为 null，前面说过 `Hashtable` 与 `HashMap` 的区别之一是它不允许 key/value 为 `null`，原因很简单：

```Java
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    ...
```

可以看到 `Hashtable.put` 直接调用 `key.hashCode()`，所以 key 当然不能为 `null` 了。

若 key 不为 `null`，则计算如下：

```Java
(h = key.hashCode()) ^ (h >>> 16)
```

为了让元素尽可能在桶中均匀分布，哈希函数要尽量离散，上述实现综合利用了 `int` 的高位和低位，避免只用低位时可能出现的哈希碰撞，具体细节还需要研究一下。

下面看下 `putVal` 的实现，这是重点，一点点看：

```Java
Node<K,V>[] tab;
Node<K,V> p;
int n, i;
```

`HashMap` 中的桶是用数组实现的，此处声明的 `tab`、`p` 都是遍历桶用的，桶声明如下：

```Java
transient Node<K,V>[] table;
```

注意 `table` 在首次使用时初始化，创建 `HashMap` 时并不会对其初始化，因此 `putVal` 首先要检查 `table` 是否初始化，若未初始化，或初始化长度为 0，则调用 `resize()` 对其扩容，并将容量赋值给 `n`：

```Java
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```

* `resize()`：若 `table` 为 `null`，则创建长度为 `threshold` 的数组，否则扩容为原来容量的两倍，容量保持为 2 的幂；

有了 key 的哈希值，如何决定该 key 在桶中的位置呢，`HashMap` 的做法很粗暴，直接取哈希值的低 `n - 1` 位，因此位置计算如下：

```Java
(n - 1) & hash
```

若该位置没有元素，则直接插入新 k-v，因为这是该桶第一个节点，因此 `next` 节点为 `null`：

```Java
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

若该位置已经有元素存在，说明该桶非空，此时 `p` 为桶中链表的表头，下一步需要在链表中查找该 key 插入的位置，首先声明一个节点来表示该 k-v 的目标位置，以及一个类型为 `K` 的临时变量：

```Java
Node<K, V> e;
K k;
```

若头结点 `p` 的哈希值、键值与该 k-v 相同，表明 `p` 就是该 k-v 的目标位置：

```Java
// 哈希值、键引用、键值都相等
if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
    e = p;
```

若头结点 `p` 是 `TreeNode`，表明链表已经进化为树，调用 `putTreeVal` 执行插入：

```Java
else if (p instanceof TreeNode)
    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```

* 为啥不先判断是否是树？为啥放在第二个检查？？？

若以上条件都不满足，则需要在链表中查找 k-v 的位置：

```Java
else {
    for (int binCount = 0; ; ++binCount) {
        // 1. 到达链表尾部
        if ((e = p.next) == null) {
            p.next = newNode(hash, key, value, null);
            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                treeifyBin(tab, hash);
            break;
        }
        // 2. 找到 key 所在元素
        if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
            break;
        p = e;
    }
}
```

若到达链表尾部，则将 k-v 插入尾部并退出循环：

```Java
if ((e = p.next) == null) {
    p.next = newNode(hash, key, value, null);
    break;
}
```

同时通过 `binCount` 循环累加，可以获得桶中的元素数量（不一定是总数），若到达链表尾部时，桶中元素数量超过 `TREEIFY_THRESHOLD - 1`，表明元素过多，此时链表进化为红黑树：

```Java
if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
    treeifyBin(tab, hash);
```

若未达到尾部，但当前元素 `e` 的哈希码、引用值（或值）相等，则表明 k-v 就应该插在 `e` 的位置上，此时直接退出循环：

```Java
if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
    break;
```

查找结束后，若 `e` 为 `null` 表明在第一个 `if` 分支 `break` 了，此时元素插入已经完成，若 `null` 不为 `null` 则表明必然是第二个 `if` 分支退出的，此时表明链表中已经存在该 `key`，此时可能需要更新旧值：

```Java
if (e != null) {  // e 不为 null 表明 HashMap 中已经存在该 key
    V oldValue = e.value;
    if (!onlyIfAbsent || oldValue == null)
        e.value = value;
    afterNodeAccess(e);
    return oldValue;
}
```

* `onlyIfAbsent` 在此处生效；

到此为止，元素插入（or 更新）已经完成，后面是一些常规操作：

```Java
++modCount;
if (++size > threshold)
    resize();
afterNodeInsertion(evict);
return null;
```

`modCount` 很有意思，表示 `HashMap` **结构化修改** 的次数，且被声明为 `transient`，用来避免对 `HashMap` 的并发修改：

```Java
transient int modCount;
```

有两类修改属于结构化修改：

* `HashMap` 映射数量的增删；
* `HashMap` 内部结构修改，如 rehash

若元素插入后数量太多，则执行扩容：

```Java
if (++size > threshold)
    resize();
```

最后，执行插入回调，并返回 `null`：

```Java
afterNodeInsertion(evict);
return null;
```

可以看到，`putVal` 中有两个地方可以返回，若 key 已存在，则返回该 key 的旧值，否则返回 `null`。

## `get`

```Java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

若 `key` 存在，则返回其映射的值，否则返回 `null`，因此返回 `null` 不能证明 `key` 不存在，也可能该 `key` 的映射值恰好为 `null`，应该用 `containsKey` 判断 key 是否存在。

实现重点在 `getNode`：

```Java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab;
    Node<K,V> first, e; 
    int n; K k;

    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null)  {
        // 该 key 可能存在
    }
    
    // 该 key 不存在
    return null;
}
```

查询时 key 可能真的存在，也可能并不存在，若满足 `table` 不为空、`table` 长度不为 0、该 key 所在桶不为空，则 key 可能已经存在：

```Java
(tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null
```

接下来检查 `key` 是否真的已经存在，首先检查桶中第一个节点的键是否为 `key`，如果是直接返回：

```Java
// always check first node
if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
    return first;
```

* 首先检查哈希值是否相同，若相同再检查键是否相等（引用相同 or `euqals` 相等）；

如果第一个节点不满足，则需要继续检查桶中剩余元素，当然前提是桶中还有元素：

```Java
if ((e = first.next) != null) {
    // 检查桶中剩余元素
}
```

桶元素可以以树或链表形式存储，如果是树，则其查找更加高效：

```Java
if (first instanceof TreeNode)
    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
```

如果是链表，则只能顺序遍历：

```Java
do {
    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
    return e;
} while ((e = e.next) != null);
```

无论是以树，还是以链表方式遍历，最后都可能找不到 `key`，此时 `getNode` 返回 `null`。

## `remove`

```Java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;
}
```

若删除成功，则返回 `key` 映射的值，否则返回 `null`，因此返回 `null` 并不能证明 `key` 不存在。

删除逻辑在 `removeNode` 中：

```Java
final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable)
```

* `matchValue` 为 `true` 则仅当 `key` 的旧值等于 `value` 时才执行删除；
* `movable` 为 `true` 则删除过程中可以移动节点，否则不移动，仅在红黑树时生效；

而 `remove(Object key)` 对 `removeNode` 调用参数如下：

```Java
removeNode(hash(key), key, null, false, true)
```

因为根本没提供 `value` 所以 `matchValue` 为 `false`，且删除过程中允许移动节点。

要删除，首先要找到 `key` 所在节点：

```Java
if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n - 1) & hash]) != null) {
    // key 可能存在，执行查找、删除
}

// 若 key 不存在
return null;
```

跟查找时相同，若 `table` 不为空、长度大于 0，且 `key` 所在桶非空时 `key` 才可能存在，此时执行查找、删除逻辑。

查找与 `getNode` 几乎相同：

```Java
Node<K,V> node = null, e; 
K k; 
V v;

// 头节点为目标节点
if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
    node = p;
else if ((e = p.next) != null) {
    // 高效的树查找
    if (p instanceof TreeNode)
        node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
    else {
        // 循环链表查找
        do {
            if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                node = e;
                break;
             }
             p = e;
         } while ((e = e.next) != null);
    }
}
```

查找完成后，若成功，则 `node` 即为目标节点，否则 `node` 为 `null`，另外 `matchValue` 还可影响删除逻辑，因此删除之前需要满足：

```Java
if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
    // 执行删除
}
```

只有 `node` 不为 `null`，且满足该结点满足删除条件时才执行删除，删除条件有两种：

```Java
!matchValue || (v = node.value) == value || (value != null && value.equals(v))
```

要么 `matchValue` 为 `false`，即不要求旧值与 `value` 相等，要么满足旧值与 `value` 相等。

删除时，如果是红黑树，则用更加高效的树删除：

```Java
if (node instanceof TreeNode)
    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
```

否则执行链表删除：

```Java
else if (node == p)
    tab[index] = node.next;
else
    p.next = node.next;
```

链表删除借助 `node` 和 `p` 实现，若 `key` 正好位于桶中第一个节点，即查找在这里结束：

```Java
if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
    node = p;
```

仅在该情况下满足 `node == p`，此时删除 `node`：

```Java
tab[index] = node.next;
```

其余场景下，`p` 指向 `node` 的前驱结点，此时删除如下：

```Java
p.next = node.next;
```

到此为止，删除操作已经完成，最后修改 `modCount`、减小 `size`、执行回调，并返回被删除的节点：

```Java
++modCount;
--size;

afterNodeRemoval(node);

return node;
```

`remove` 还有一个重载方法：

```Java
public boolean remove(Object key, Object value) {
    return removeNode(hash(key), key, value, true, true) != null;
}
```

同样委托 `removeNode` 实现删除逻辑，但因为此时同时提供了 `key` 和 `value`，所以 `matchValue` 为 `true`。

注意返回值为 `true` 表明删除成功，但我们注意到，是通过将 `removeNode` 返回值与 `null` 判断实现的，因为 `removeNode` 为 `null` 证明该 `key` 不存在。

>前面说过，`remove` 返回值为 `null` 并不能证明该 `key` 不存在。

## compute 类

Java 8 新增了 3 个 compute 类方法：

* `compute`
* `computeIfAbsent`
* `computeIfPresent`

### `compute`

```Java
public V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) 
```

从签名可以猜测，`key` 用于指定要修改的映射，`remappingFunction` 则以 `key` 及其旧值为参数，计算得到一个新的 `value`，最后将 `key` 的值修改为 `value`，`compute` 该为 `updateInPlace` 似乎更合理，本身就是原地修改。

以上只是我们的猜测，`compute` 实际行为需要走读源码才能掌握。

首先，`remappingFunction` 不能为 `null`，否则抛出空指针异常：

```Java
if (remappingFunction == null)
    throw new NullPointerException();
```

然后是需要用到的中间变量的声明：

```Java
int hash = hash(key);

Node<K,V>[] tab; 
Node<K,V> first; 
int n, i;
int binCount = 0;
TreeNode<K,V> t = null;
Node<K,V> old = null;
```

* `tab` 是指向 `table` 的引用；
* `first` 是 `key` 所在桶的首元素；
* `n` 是 `table` 的长度，即桶个数；
* `i` 是 `key` 所在桶在 `table` 中的索引，所以 `first` 就是 `tab[i]`；
* `binCount` 是桶中元素的数量；
* `t` 若桶中链表退化为树，则 `t` 为首元素 `first` 转换为 `TreeNode` 后的值；
* `old` 是 `key` 所在节点；

实际查找 `key` 所在节点之前，先检查 `table` 是否需要扩容，或是否需要初始化：

```Java
if (size > threshold || (tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```

注意，先检查是否需要扩容，即 `size > threshold`，若放在后面，则只能执行一次，之后 `n` 赋值为 `table` 的长度。

准备工作做好后，查找 `key` 所在节点：

```Java
if ((first = tab[i = (n - 1) & hash]) != null) {
    if (first instanceof TreeNode)
        // 树查找
    else {
        // 链表查找
    }
}
```

首先判断桶中头元素是否为空，如果是空，则 `key` 肯定不存在，若首元素 `first` 为 `TreeNode`，则执行树查找：

```Java
old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
```

否则执行链表查找，并统计桶中元素个数：

```Java
Node<K,V> e = first; 
K k;
do {
    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
        old = e;
        break;
    }
    ++binCount;
} while ((e = e.next) != null);
```

查找过程很简单，即顺序遍历并比较，若查找成功，则 `old` 指向 `key` 所在节点，否则 `old` 为 `null`。

然后需要根据 `key` 所在节点的映射值和 `remappingFunction` 计算新的值，因为 `key` 不一定真的存在，若 `key` 不存在怎么办呢，直接返回码？当节点不存在时，`HashMap` 实际将旧值视为 `null`：

```Java
V oldValue = (old == null) ? null : old.value;
```

新值计算很直接：

```Java
V v = remappingFunction.apply(key, oldValue);
```

因为 `key` 可能并不存在，所以更新值分为两种场景：

```Java
if (old != null) {
    // key 已存在
}
else if (v != null) {
    // key 不存在，且新值不为 null
}
```

若 `key` 已经存在，且新值不为 `null`，则更新已有节点的值，若新值为 `null`，则删除该节点：

```Java
if (v != null) {
    old.value = v;
    afterNodeAccess(old);
}
else
    removeNode(hash, key, null, false, true);
```

若 `key` 不存在，且新值不为 `null`，则插入新节点：

```Java
if (t != null)
    t.putTreeVal(this, tab, hash, key, v);
else {
    tab[i] = newNode(hash, key, v, first);
    if (binCount >= TREEIFY_THRESHOLD - 1)
        treeifyBin(tab, hash);
}
++modCount;
++size;
afterNodeInsertion(true);
```

注意，若是链表插入，则将新节点插在表头，且还需要检查是否需要 `treeifyBin`。

若 `key` 不存在，且新值为 `null` 呢，代码没有处理该场景，而是直接走到最后：

```Java
return v;
```

可以看到，`key` 不存在，且新值为 `null` 时，不插入新节点，且返回 `null`。

总结一下：

* `key` 存在
  + 新值不为 `null`：更新节点
  + 新值为 `null`：删除节点
* `key` 不存在：
  + 新值不为 `null`：插入节点
  + 新值为 `null`：啥都不干

如此看来，叫 `updateInPlace` 也不合理，因为该函数会做更新、删除、插入 3 中操作，叫 `compute` 那是非常贴切的。

`compute` 在 `key` 存在或不存在时都能工作，但有时需明确区分这两种场景，因此 Java 8 还提供了：

* `computeIfAbsent`
* `computeIfPresent`

### `computeIfAbsent`

顾名思义，只有 `key` 不存在时才计算，因此 `computeIfAbsent` 签名如下：

```Java
public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) 
```

### `computeIfPresent`

// TODO

## `clear`

```Java
// Removes all of the mappings from this map. The map will be empty after this call returns.
public void clear() {
    Node<K,V>[] tab;
    modCount++;
    if ((tab = table) != null && size > 0) {
        size = 0;
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```

`clear` 属于结构性修改，因此 `modCount++`，通过将每个桶的 **头元素** 设为 `null` 实现清理：

```Java
for (int i = 0; i < tab.length; ++i)
    tab[i] = null;
```

注意，并非将 `HashMap` 中所有元素设置 `null`，而是仅设置头元素，这么做肯定能达到清空 `HashMap` 的效果，有人会问，为啥不直接把 `table` 置为 `null` 呢，不是更简单？

`table = null` 也能达到清理的效果，但 `clear` 的设计使用场景是：移除已有映射后，还想 **复用** 该 `HashMap`，且新填充的映射数量与之前几乎相同。因此 `clear` 实现没有粗暴的将 `table` 设为 `null`，参见[该回答](https://stackoverflow.com/questions/2811537/is-java-hashmap-clear-and-remove-memory-effective)。

如果不需要复用 `HashMap`，或之后插入的映射数量相对之前 **很少**，则不推荐使用 `clear`，而是：

```Java
m = null;
HashMap n = new HashMap<>();
```

即将原 `HashMap` 设为 `null`，之后 GC 将回收它占用的内存，并新建 `HashMap` 使用。

## `clone`

```Java
public Object clone() {
    HashMap<K,V> result;
    try {
        result = (HashMap<K,V>)super.clone();
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
    result.reinitialize();
    result.putMapEntries(this, false);
    return result;
}
```

`clone` 方法的注释写到：

```
Returns a shallow copy of this HashMap instance: the keys and values themselves are not cloned.
```

即 `HashMap.clone` 实现的是浅克隆，首先调用 `AbstractMap.clone`：

```Java
result = (HashMap<K,V>)super.clone();
```

`AbstractMap.clone` 也是浅克隆：

```Java
protected Object clone() throws CloneNotSupportedException {
    AbstractMap<?,?> result = (AbstractMap<?,?>)super.clone();
    result.keySet = null;
    result.values = null;
    return result;
}
```

然后对克隆结果 `result` 重新初始化：

```Java
// Reset to initial default state.  Called by clone and readObject.
void reinitialize() {
    table = null;
    entrySet = null;
    keySet = null;
    values = null;
    modCount = 0;
    threshold = 0;
    size = 0;
}
```

`reinitialize` 将 `result` 全部状态字段设置为空，包括桶（`table`）、集合视图（`entrySet`、`keySet`、`values`），以及各种计数字段，重新初始化后，`result` 剩下的状态字段都是 `final` 的，比如 `loadFactor` 等。

最后，使用 `HashMap.putMapEntries` 将当前 map（`this`）的所有映射插入 `result`，开头根据当前 map 的大小（`m.size`）将 `result` 容量初始化为合理大小：

```Java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
            resize();
        // 循环插入 k-v
    }
}
```

为消除批量插入过程中的扩容，需计算需要的最小容量：

```Java
float ft = ((float)s / loadFactor) + 1.0F;
int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);
```

若最小容量大于 `threshold`，表明插入时肯定需要扩容，此时将 `threshold` 设置为 `t`，循环插入时，`putVal` 第一次扩容就直接扩到 `t`，省去多次扩容的开销：

```Java
if (t > threshold)
    threshold = tableSizeFor(t);
```

如果 `t` 小于 `threshold`，表明插入时不会发生扩容，那我们也不用多此一举。

还有一种情况，若 `table` 不为 `null`，但 `s > threshold`，`clone` 调用时该场景不会发生，但其他场景可能发生，此时 **立即** 扩容一次：

```Java
else if (s > threshold)
    resize();
```

容量准备好后，就循环将 `m` 中的映射插入 `result`：

```Java
for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
    K key = e.getKey();
    V value = e.getValue();
    putVal(hash(key), key, value, false, evict);
}
```

## contains 类

`HashMap` 中保存的是 key-value 映射，因此 `HashMap` 分别为键、值提供了检测方法：

* `containsValue`
* `containsKey`

### `containsKey`

```Java
public boolean containsKey(Object key) {
    return getNode(hash(key), key) != null;
}
```

key 检测借助 `getNode` 实现，如果能查找到该 key 的节点，则表明该 key 存在，非常明确。

### `containsValue`

检测 key 很简单，直接查找即可，检测 value 没有现成方法，只能自行实现。

首先，只有 `table` 不为空，且长度不为 0 时 value 才可能存在：

```Java
Node<K,V>[] tab; 
V v;

if ((tab = table) != null && size > 0) {
    // 可能存在 value
}

return false;
```

查找时，先遍历桶，然后遍历桶中元素：

```Java
for (int i = 0; i < tab.length; ++i) {
    for (Node<K,V> e = tab[i]; e != null; e = e.next) {
        if ((v = e.value) == value || (value != null && value.equals(v)))
            return true;
    }
}
```

若查找成功，返回 `true`，查找失败则返回 `false`。

## 集合视图

`HashMap` 提供 3 种集合视图：

* `keySet`
* `values`
* `entrySet`

### `keySet`

`HashMap` 对 `keySet` 实现很简单，若 `keySet` 不为空，则返回 `keySet`，否则返回空 `KeySet`：

```Java
public Set<K> keySet() {
    Set<K> ks = keySet;
    if (ks == null) {
        ks = new KeySet();
        keySet = ks;
    }
    return ks;
}
```

而 `keySet` 定义在 `AbstractMap` 中：

```Java
transient Set<K>        keySet;
```

`AbstractMap` 和 `HashMap` 对 `keySet` 做赋值的地方只有一处，就是上面 `if` 分支：

```Java
ks = new KeySet();
```

因此，关键在于 `KeySet` 类：

```Java
final class KeySet extends AbstractSet<K> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<K> iterator()     { return new KeyIterator(); }
    public final boolean contains(Object o) { return containsKey(o); }
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    public final Spliterator<K> spliterator() {
        return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super K> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

可以看到 `KeySet` 所有方法（比如 `size`、`contains`、`remove` 等）都是基于 `HashMap` 的底层存储 `table` 实现的，因此如果 `HashMap` 的数据发生变化，`KeySet` 立马会被影响：

```Java
HashMap<String, Integer> m = new HashMap<>();
m.put("a", 1);
m.put("b", 2);

Set<String> keys = m.keySet();
System.out.println(keys);  // [a, b]
        
m.remove("a");
System.out.println(keys);  // [b]
```

反过来，因为 `KeySet.remove` 是通过修改 `HashMap` 的 `table` 实现的，因此删除键也会立马影响 `HashMap`：

```Java
HashMap<String, Integer> m = new HashMap<>();
m.put("a", 1);
m.put("b", 2);

Set<String> keys = m.keySet();
System.out.println(m);  // {a=1, b=2}

keys.remove("a");
System.out.println(m);  // {b=2}
```

当然，并非只能通过源码才能知道 `KeySet` 和 `HashMap` 互相影响，`keySet` 方法的 javadoc 里实际早就明确指出：

```
Returns a Set view of the keys contained in this map. The set is backed by the map, 
so changes to the map are reflected in the set, and vice-versa.  

If the map is modified while an iteration over the set is in progress (except through the 
iterator's own remove operation), the results of the iteration are undefined.  

The set supports element removal, which removes the corresponding mapping from the map, 
via the Iterator.remove, Set.remove, removeAll, retainAll, and clear operations.  
It does not support the add or addAll operations.
```

上面还说如果 `HashMap` 在遍历 `KeySet` 过程中被修改，则遍历结果未定义，啥意思呢？无非就是部分元素遍历不到，or 遍历到不一致的数据，我们看下原因。

`KeySet` 提供两种遍历方式，即：

* `forEach`
* `iterator`

先看下 `forEach`：

```Java
public final void forEach(Consumer<? super K> action) {
    Node<K,V>[] tab;
    if (action == null)
        throw new NullPointerException();
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next)
                action.accept(e.key);
        }
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```

如果 `forEach` 遍历过程中 `HashMap` 有结构化修改，则抛出 `ConcurrentModificationException`，这是通过在遍历前（嵌套 `for`）记录 `modCount`，并在遍历后与当前 `modCount` 比较实现的：

```Java
int mc = modCount;
// 遍历
if (modCount != mc)
    throw new ConcurrentModificationException();
```

再看下 `iterator`：

```Java
public final Iterator<K> iterator()     { return new KeyIterator(); }
```

`iterator` 返回一个 `KeyIterator`，通过该 `Iterator` 可以实现 key 遍历，`KeyIterator` 实现如下：

```Java
final class KeyIterator extends HashIterator implements Iterator<K> {
    public final K next() { return nextNode().key; }
}
```

`KeyIterator.next` 是委托给 `HashIterator.nextNode` 实现的，因此重点在 `HashIterator`，这里说句题外话，`HashMap` 有 3 个分别用于 key/value/entry 的遍历器：

```Java
final class KeyIterator extends HashIterator implements Iterator<K> {
    public final K next() { return nextNode().key; }
}

final class ValueIterator extends HashIterator implements Iterator<V> {
    public final V next() { return nextNode().value; }
}

final class EntryIterator extends HashIterator implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}
```

他们都是 `HashIterator` 的子类，且大部分功能都委托给 `HashIterator` 实现，因此理解 `HashIterator` 对于理解 `HashMap` 遍历非常重要。

`HashIterator` 是一个抽象类，共有 4 个状态字段和 3 个方法：

```Java
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    public final boolean hasNext() { }
    final Node<K,V> nextNode() { }
    public final void remove() { }
}
```

4 个字段分别用于：

* `index` 当前桶索引，`current` 当前 k-v 引用
* `next` 下一个 k-v
* `expectedModCount` 为 `modCount` 的副本

构造时初始化所有字段：

```Java
HashIterator() {
    expectedModCount = modCount;
    Node<K,V>[] t = table;
    current = next = null;
    index = 0;
    if (t != null && size > 0) { // advance to first entry
        do {} while (index < t.length && (next = t[index++]) == null);
    }
}
```

`expectedModCount` 和 `current` 初始化比较简单，当前节点默认为 `null`，因此 `current` 被初始化为 `null`。

关键一句是 `next = t[index++]`，这句完成 `next` 和 `index` 的初始化，最后 `index` 为第一个不为空的桶的索引，而 `next` 为该桶首元素的引用。

`hasNext` 用于判断是否还有剩余元素，`next` 不为空则表示还有剩余：

```Java
public final boolean hasNext() {
    return next != null;
}
```

`nextNode` 返回下一个元素，首先判断 `HashMap` 是否有结构性修改，以及是否已经没有元素了：

```Java
final Node<K,V> nextNode() {
    Node<K,V>[] t;
    Node<K,V> e = next;

    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    if (e == null)
        throw new NoSuchElementException();
}
```

返回下个节点容易，直接把 `e` 返回即可，但返回后需要修改 `HashInterator` 的内部状态，即 `current`、`next` 和 `index`，这是 `nextNode` 正确工作的关键：

```Java
if ((next = (current = e).next) == null && (t = table) != null) {
    do {} while (index < t.length && (next = t[index++]) == null);
}
```

`next = (current = e).next` 很巧妙，完成了 `next` 和 `current` 移动，移动后若 `next` 为 `null`，且 `table` 不为 `null`，此时若无其他动作，下次 `nextNode` 调用将返回 `null`，但 `HashMap` 还不致于如此弱智，因为 `next` 和 `current` 在同一个桶中，所以 `next` 为 `null` 表明该桶已经遍历结束，此时需要前往下一个非空的桶：

```Java
do {} while (index < t.length && (next = t[index++]) == null);
```

这里为啥不直接用 `while`，而是用 `do while` 呢？我也没搞明白，我觉得语义上没有区别，可能虚拟机实现上有区别吧。

`remove` 实现也比较简单，首先还是各种检查：

```Java
public final void remove() {
    Node<K,V> p = current;

    if (p == null)
        throw new IllegalStateException();
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();

    ...
}
```

核心的删除逻辑是通过 `removeNode` 实现的，删除之前先将 `current` 置为 `null`，便于垃圾回收，然后执行删除：

```Java
current = null;
K key = p.key;
removeNode(hash(key), key, null, false, false);
```

注意，前面说过遍历 `KeySet` 时若 `HashMap` 发生结构化修改，则抛出 `ConcurrentModificationException`，但通过 `Iterator.remove` 删除除外，原因是 `Iterator.remove` 在删除元素后修改了 `expectedModCount`：

```Java
expectedModCount = modCount;
```

这样遍历时 `nextNode` 中的检查就能通过了：

```Java
final Node<K,V> nextNode() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
```

### `values`

`keySet()` 返回键的集合，而 `values` 返回值的集合，不过因为可能存在多个相同值，所以返回的集合类型为 `Collection`：

```Java
public Collection<V> values() {
    Collection<V> vs = values;
    if (vs == null) {
        vs = new Values();
        values = vs;
    }
    return vs;
}
```

实现与 `keySet` 几乎完全相同，同样 `Values` 与 `HashMap` 修改时互相影响，且遍历 `values()` 结果时，若 `HashMap` 发生结构性修改（通过 `Iterator.remove` 删除除外），则抛出 `ConcurrentModificationException`。

同样，`values` 重点也在 `Values` 类：

```Java
final class Values extends AbstractCollection<V> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<V> iterator()     { return new ValueIterator(); }
    public final boolean contains(Object o) { return containsValue(o); }
    public final Spliterator<V> spliterator() {
        return new ValueSpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.value);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

遍历 `Values` 通过 `ValueIterator` 实现：

```Java
final class ValueIterator extends HashIterator implements Iterator<V> {
    public final V next() { return nextNode().value; }
}
```

`HashIterator` 前面已经介绍过了。

### `entrySet`

`keySet` 和 `values` 各从键、值两个角度反映了 `HashMap` 的内部结构，`entrySet` 则是从键值映射角度：

```Java
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
```

这次代码比 `keySet` 和 `values` 要简洁一些，汗。

`Set<Map.Entry<K, V>>` 与 `HashMap` 同样互相影响，结构化修改同样会抛出 `ConcurrentModificationException`，但例外场景有两个：

* `Iterator.remove`
* `Entry.setValue`

这两个方法修改不会导致 `ConcurrentModificationException`。

实现重点同样在 `EntrySet`，与 `KeySet` 和 `Values` 大同小异。

## 回调函数

`HashMap` 提供以下 3 个回调函数：

```Java
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

`HashMap` 中这 3 个方法为空实现，主要留给 `LinkedHashMap`（`LinkedHashMap` 继承 `HashMap`）使用。

## 元素查找

元素查找即查找指定 key 所在的节点，可以分为两步：

1. 桶查找
2. 链表/红黑树查找

### 桶查找

`HashMap` 中的桶是一个 `Node<K, V>[] table` 数组，给定 `key-value`，通过如下公式确定该 `key` 所在的桶的索引：

```Java
(n - 1) & hash
```

由于 `n` 是 2 的整数幂，因此 `n - 1` 正好是只保留 `hash` 的 `n - 1` 位的掩码，例如 n = 16 时，n - 1 的二进制表示为：

```
1111
```

而 `1111 & hash` 的效果是过滤掉低 4 位以上的其他字段，最后的结果可以直接作为桶的索引。

其效果与 `hash % n` 相同（前提：n 是 2 的整数幂），但是位操作比 `%` 更加高效。

### 链表/红黑树查找

找到 key 所在桶后，桶中元素要么形成链表，要么形成红黑树，无论哪种情况，要找到 key 所在节点，需：

```Java
if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
```

判断节点 `e` 的键是否为 `key`，首先要满足哈希值相等，然后满足键相等，键相等分为两种情况：

* 引用相同，表明两个键实际指向同一个对象，自然相等；
* 引用不同，但 `equals` 相等，表明两个键对象在 `equals` 意义上相等；

可以看出，如果要把类型 `A` 作为 `HashMap` 的键使用，则必须保证类型 `A` 正确实现了 `equals` 方法（实际还需要实现 `hashCode` 方法），否则无法保证能查找到已插入的映射。

还有一点，如果 `key` 为 `null`，且当前节点 `e` 的键也是 `null`，则根据 `hash` 函数可知，`hash(null)` 为 0，因此满足 `e.hash == hash`，而 `null == null` 结果也是 `true`，因此 `null` 键也可以被正确查找。

## `HashMap` 在并发场景中问题

`HashMap` 非并发安全，但并非并发场景一定不能用 `HashMap`，只不过需要做额外同步，相对直接用 `ConcurrentHashMap` 就有点得不偿失了，了解下 `HashMap` 在并发场景下存在的问题还是有点意思的。

>若所有线程对 `HashMap` 都是读取操作，则无需同步。

### 丢失元素

即 `put` 时，不同 key 同时插入，但因为 `putVal` 未做同步，导致插入位置相同：

```Java
if ((e = p.next) == null) {
    p.next = newNode(hash, key, value, null);
}
```

>此处注意，Java 7 在头部插入，Java 8 在尾部插入。

多个线程都在同一个链表尾部插入，肯定只有最后一个的结果能保存，其他的都丢失了。

### `get` 死循环

网上很多文章提到该问题，Java 8 已经修复。
