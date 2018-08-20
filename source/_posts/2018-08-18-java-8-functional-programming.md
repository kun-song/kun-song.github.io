---
title: Java 8 Stream 入门
date: 2018-08-18 12:24:17
tags:
  - Java
  - fp
categories: 技术
---

面向对象编程是对 **数据** 进行抽象，函数式编程是对 **行为** 进行抽象，现实世界中数据与行为并存，为不同的任务选择合适的抽象方式，两全其美。

通常而言，函数式代码在 **效率** 上不是最优的，但可读性比面向对象代码好，如今处理器性能已经过剩，若不是特别关注效率的场景，则可读性优先。

虽然 Java 8 对 FP 支持偏薄弱，例如不支持惰性求值、不可变数据结构、typeclass 等，但相比以前，已经大大进步了。

<!-- more -->

## lambda 表达式

为了实现类似 Scala 的超级好用的集合组合子，Java 必须在语言层面变革：新增 lambda 表达式。

例如：

```Java
Runnable task = () -> System.out.println("Hello");
BinaryOperator<Long> add = (x, y) -> x + y;
```

* `task` 类型为 `Runnable`
* `add` 类型为 `BinaryOperator<Long>`

### `final` 变量

lambda 表达式中引用的局部变量必须是：

* `final` 变量，或
* 事实上的 `final` 变量

`final` 变量很好理解，实际上的 `final` 变量即 **只赋值过一次** 的变量，例如：

```Java
int n = 10;
Runnable task = () -> System.out.println(n);
```

若再次为 `n` 赋值：

```Java
int n = 10;
n = 20;
Runnable task = () -> System.out.println(n);
```

编译器提示：

```Java
Variable used in lambda expression should be final or effectively final
```

### 函数接口

>函数接口是只有一个 **抽象方法** 的接口，用作 lambda 表达式的类型。

`BiFunction` 即为函数接口，因为它只有 `apply` 一个抽象方法（abstract method），`andThen` 是已经实现的默认方法：

```Java
@FunctionalInterface
public interface BiFunction<T, U, R> {
    R apply(T t, U u);

    default <V> BiFunction<T, U, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t, U u) -> after.apply(apply(t, u));
    }
}
```

* `BiFunction` 是接口，所以不需要用 `abstract` 修饰 `apply` 方法；

函数接口的唯一抽象方法的 **名字并不重要**，一般不会直接使用，习惯将其命名为 `apply`，其参数类型、返回类型比较重要。

JDK 常见函数接口如下：

* `Predicate<T>` 类型为 `T => Boolean`
* `Consumer<T>` 类型为 `T => Void`
* `Function<T, R>` 类型为 `T => R`
* `Supplier<T>` 类型为 `() => T`
* `UnaryOperator<T>` 类型为 `T => T`
* `BinaryOperator<T, R>` 类型为 `(T, T) => T`

### 例子

不少类通过 lambda 表达式得到了增强，例如若要创建具备初始值的 `ThreadLocal` 实例，以前需要：

```Java
ThreadLocal t = new ThreadLocal() {
    @Override
    protected Object initialValue() {
        return 10;
    }
};
```

Java 8 可以用 `Supplier` 创建，更加易读：

```Java
ThreadLocal t = ThreadLocal.withInitial(() -> 10);
```

看下 `withInitial` 的实现：

```Java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}
```

而 `SuppliedThreadLocal` 则覆盖了默认的 `initialValue` 方法：

```Java
/**
 * An extension of ThreadLocal that obtains its initial value from
 * the specified {@code Supplier}.
 */
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```

## 流

Java 8 通过 lambda 表达式大大提升了集合操作的可用性，包括：

* 集合 API 增强
* `Stream`

### 外部迭代 vs 内部迭代

传统的 `for` 迭代存在以下缺点：

* 不易并行
* 语法噪音影响理解，对于复杂、嵌套的迭代尤其明显

`for` 循环是通过 **外部迭代** 实现的，即首先调用集合的 `interator()` 方法产生一个 `Iterator` 对象，然后通过该对象的 `hasNext` 和 `next` 方法控制迭代过程：

```Java
int count = 0;
Iterator<Artist> iterator = allArtists.iterator(); 
while(iterator.hasNext()) {
    Artist artist = iterator.next(); 
    if (artist.isFrom("London")) {
        count++;
    }
}
```

外部迭代过程如下图：

<img src="/images/java-8-lambdas/outer-iteration.png" alt="外部迭代" style="width: 400px;"/>

Java 8 添加的 `stream()` 方法类似 `iterator()`，但返回的不是 `Iterator` 对象，而是用于内部迭代的 `Stream` 接口：

```Java
long count = allArtists.stream()
                       .filter(artist -> artist.isFrom("London"))
                       .count();
```

内部迭代过程如下图：

<img src="/images/java-8-lambdas/inner-iteration.png" alt="外部迭代" style="width: 400px;"/>

### 实现机制

`Stream` 方法有两种求值策略：

* 返回值为 `Stream` 的方法，如 `filter`，是 lazy evaluation
* 返回值非 `Stream`（包含 `void`），如 `count`，是 eager evalution

使用 `Stream` 的一般模式是，创建一组 lazy 求值方法组成的调用链，最后用一个 eager 求值方法终结之，整个调用链将只对集合迭代一次。

### 常用函数

#### `collect(toList())`

`collect` 非常强大，生成 `List` 仅仅是最简单的应用：

```Java
List<String> collected = Stream.of("a", "b", "c")
                               .collect(Collectors.toList());
```

#### `map`

```Java
List<String> collected = Stream.of("a", "b", "hello")
                               .map(string -> string.toUpperCase()) 􏰐
                               .collect(toList());
```

#### `filter`

```Java
List<String> beginningWithNumbers
       = Stream.of("a", "1abc", "abc1")
               .filter(value -> isDigit(value.charAt(0)))
               .collect(toList());
```

#### `flatMap`

```Java
List<Integer> together = Stream.of(asList(1, 2), asList(3, 4))
                               .flatMap(numbers -> numbers.stream())
                               .collect(toList());
```

#### `max` 和 `min`

```Java
List<Track> tracks = asList(new Track("Bakai", 524), new Track("Violets for Your Furs", 378), new Track("Time Was", 451));

Track shortestTrack = tracks.stream()
                            .min(Comparator.comparing(track -> track.getLength()))
                            .get();
```

* `max` 和 `min` 返回值都是 `Optional`；

#### `reduce`

`reduce` 将一组值归约（reduce）为一个值，`count`、`max` 和 `min` 都是 `reduce` 的特例，因为过于常见，标准库将其单独定义。

利用 `reduce` 求和：

```Java
int sum = Stream.of(1, 2, 3)
                .reduce(0, Integer::sum);
```

* 该求和方式仅为演示 `reduce` 用法，生产环境不要用！

#### 返回 `Stream` 的方法

相对返回 `List` 或 `Set`，返回 `Stream` 仅仅暴露了 `Stream` 接口，用户无法通过返回的 `Stream` 修改背后的集合！

#### 多次调用 `Stream` 操作

```Java
List<Artist> musicians = album.getMusicians()
                              .collect(toList());
List<Artist> bands = musicians.stream()
                              .filter(artist -> artist.getName().startsWith("The"))
                              .collect(toList());
Set<String> origins = bands.stream()
                           .map(artist -> artist.getNationality())
                           .collect(toSet());
```

以上代码缺点：

1. 可读性差，样板代码太多，隐藏了真实的代码意图；
1. 效率差，每一步都强制计算，生成新集合；
1. 垃圾中间变量；
1. 难以并行化；

改为：

```Java
Set<String> origins = album.getMusicians()
                           .filter(artist -> artist.getName().startsWith("The"))
                           .map(artist -> artist.getNationality())
                           .collect(toSet());
```

#### 练习

用 `reduce` 实现 `filter`：

```Java
private static <T> List<T> filter(Stream<T> xs, Predicate<T> p) {
  BiFunction<List<T>, T, List<T>> accumulator =
  (acc, x) -> {
    if (p.test(x)) {
      List<T> newAcc = new ArrayList<>(acc);
      newAcc.add(x);
      return newAcc;
    }
    else return acc;
  };
BinaryOperator<List<T>> combiner =
  (zs, ys) -> {
    List<T> result = new ArrayList<>(zs);
    result.addAll(ys);
    return result;
  };

return xs.reduce(Collections.emptyList(), accumulator, combiner);
}

// List(3, 4)
List<Integer> xs = filter(Stream.of(1, 2, 3, 4), n -> n > 2);
```

用 `reduce` 实现 `map`：

```Java
private static <T, R> List<R> map(Stream<T> xs, Function<T, R> mapper) {
  BiFunction<List<R>, T, List<R>> accumulator =
    (acc, x) -> {
      List<R> newAcc = new ArrayList<>(acc);
      newAcc.add(mapper.apply(x));
      return newAcc;
    };
  BinaryOperator<List<R>> combiner =
    (zs, ys) -> {
      List<R> result = new ArrayList<>(zs);
      result.addAll(ys);
      return result;
    };

  return xs.reduce(Collections.emptyList(), accumulator, combiner);
}

// List(2, 3, 4, 5)
List<Integer> ys = map(Stream.of(1, 2, 3, 4), n -> n + 1);
```

## 类库

### 原始类型

`int` 作为基本类型，有与之对应的装箱类型 `Integer`：

* 两者内存占用不同（`int` 4 字节，`Integer` 16 字节），最坏情况下，同样大小的数组，`Integer[]` 比 `int[]` 多占用 6 倍内存；
* 装箱、拆箱还需要额外的计算开销；

对于需要 **大量数值计算** 的算法，内存占用 + 装箱、拆箱将明显减缓程序的运行速度，为减缓这些性能开销，`Stream` 对原始类型和装箱类型做了显式区分，目前仅对：

* `int`
* `long`
* `double`

做了特殊处理，因为它们在数值计算中使用最为广泛。

针对原始类型做特殊处理的方法在命名上有 **明确规范**：

* 若 **返回类型** 为原始类型，则加 `to`，如 `ToLongFunction`；
* 若 **参数** 为原始类型，则不加前缀，如 `LongFunction`；
* 若高阶函数使用原始类型，则加 `to` 加原始类型名，如 `mapToLong`。

Java 8 为原始类型准备了与之对应的 stream，如 `IntStream`、`LongStream` 等，而 `mapToLong` 返回的结果是 `LongStream`：

```Java
LongStream a = Stream.of("Hello", "world!").mapToLong(String::length);
```

`LongStream` 中的很多高阶函数实现与普通 `Stream` 不同，例如 `map`：

```Java
LongStream b = a.map(l -> l + 1);
```

推荐尽量使用 `IntStream`、`DoubleStream` 等特殊处理的 stream，除更好的性能外，`IntStream` 等还有一些额外的用于数值计算的方法，如：

```Java
LongStream a = Stream.of("Hello", "world!").mapToLong(String::length);
// LongSummaryStatistics{count=2, sum=11, min=5, average=5.500000, max=6}
LongSummaryStatistics statistics = a.summaryStatistics();
```

`summaryStatistics` 方法可计算多种统计值，若不需要全部统计值，也可分别获取：

```Java
OptionalLong max = a.max();
OptionalLong min = a.min();
long count = a.count();
long sum = a.sum();
OptionalDouble average = a.average();
```

### 重载解析

Java 允许方法重载，即方法名字相同，但签名不同，方法重载将影响类型推断，一般而言，Java 编译器将选择 **最具体** 的类型。

现有两个 `overloadedMethod` 方法：

```Java
private void overloadedMethod(Object o) { System.out.print("Object");
}
private void overloadedMethod(String s) { System.out.print("String");
}
```

方法调用 `overloadedMethod("abc")` 将选择第二个，因为参数 `String` 更加具体，且满足推断条件。

对于 lambda 表达式也是一样，lambda 表达式的类型为对应的 **接口类型**：

```Java
private interface IntegerBiFunction extends BinaryOperator<Integer> {
}
private void overloadedMethod(BinaryOperator<Integer> Lambda) { System.out.print("BinaryOperator");
}
private void overloadedMethod(IntegerBiFunction Lambda) {
         System.out.print("IntegerBinaryOperator");
     }
```

方法调用 `overloadedMethod((x, y) -> x + y)` 将选择 **更具体** 的 `IntegerBiFunction`。

但有时编译器无法推断出最具体的类型，例如：

```Java

private interface IntPredicate { public boolean test(int value);
}
private void overloadedMethod(Predicate<Integer> predicate) { System.out.print("Predicate");
}
private void overloadedMethod(IntPredicate predicate) { System.out.print("IntPredicate");
}
```

方法调用 `overloadedMethod((x) -> true)` 无法找到 `IntPredicate` 和 `Predicate` 之间更具体的那个类型，因为这两者对于该调用”同样具体“，此时编译器将报错，解决方法是把 lambda 表达式强制转换为 `IntPredicate` 或 `Predicate`。

>实际上，该现象属于”代码异味“，此时不应该重载 `overloadedMethod`，应该重新命名。

### `@FunctionalInterface`

前面说过函数接口是只有一个抽象方法的接口，但抽象方法仅仅是函数接口的必要不充分条件。

有的接口恰好只有一个抽象方法，但这并不意味它是为 lambda 表达式设计的，例如 `Comparable` 和 `Closeable`，用 lambda 表达式实现两者几乎总是无意义的，因为：

* 一般不认为函数之间存在顺序（`Comparable`）；
* 一个操作外部资源，且可能抛出异常的方法（`public void close() throws IOException`）也不适合实现为 lambda 表达式；

作为对比，为提升 `Stream` 可操作性引入的各种新接口（`Predicate` 等）都需要用 lambda 表达式实现，因此用 `@FunctionalInterface` 修饰，防止无意破坏函数接口的条件。

### 向后兼容

Java 一直保持向后兼容，即 Java 1-7 编译的类，可以直接运行在 Java 8 上。

Java 8 对集合框架进行了大量修改，例如为 `Collection` 接口添加了 `stream()` 方法，为实现向后兼容，所有实现 `Collection` 接口的类都必须新增 `stream` 方法，作为 JDK 来讲这问题不大，重新实现这些方法即可；但对于第三方的集合库，比如 `MyList`，无法强制它新增 `stream` 方法，因此它们在 Java 8 上无法运行，从而打破向后兼容。

### 默认方法

为解决向后兼容，Java 8 新增 **默认方法** 特性，如果子类没有重新实现默认方法，则直接使用父类的默认方法。

例如 `Collection.stream` 被实现为默认方法，若 `MyList` 没有实现 `stream`，则可以使用 `Collection` 的默认实现即可，所以 `MyList` 也可以运行在 Java 8 上了。

`Iterable` 也新增了一个默认方法 `foreach`：

```Java
public interface Iterable<T> {
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
}
```

#### 默认方法和子类

// TODO

### 多重继承

接口允许多重继承，因此可能遇到两个父接口包含签名相同的默认方法，例如：

```Java
public interface Jukebox {
    public default String rock() {
        return "... all over the world!";
    } 
}
public interface Carriage {
    public default String rock() {
        return "... from side to side";
    }
}

public class MusicalCarriage implements Carriage, Jukebox { }
```

此时编译器无法明确应该继承哪个 `rock`，因此会报错。在子类中实现 `rock` 即可解决：

```Java
public class MusicalCarriage implements Carriage, Jukebox {
    @Override
    public String rock() {
        return Carriage.super.rock();
    }
}
```

注意 `Carriage.super.rock()` 用法，Java 8 增强了 `super` 语法，可以指定调用哪个父接口中的方法。

#### 三定律

// TODO

### 权衡

Java 不支持类的多重继承，但支持接口的多重继承，现在接口支持 **方法实现**，实际上接口已经支持类多重继承的部分功能。Java 原本特意避免类的多重继承，因为多重继承会导致很多问题，其中更根本的原因是 **状态** 的多重继承。

因为接口没有状态（字段），只有行为（方法），因此接口多重继承避免了臭名昭著的 **状态** 继承，也就避免了多重继承的最大问题。

既然接口可以定义方法实现，那它与抽象类还有什么区别吗？

主要区别：

* 接口允许多重继承，但没有成员变量；
* 抽象类不支持多重继承，但有成员变量；

### 接口的静态方法

以前经常把静态方法放到工具类中，但若某静态方法与某个概念强相关，那把该方法与相关类、接口放到一起更加合理。

类自然可以有 `static` 方法，而 Java 8 开始，接口也可以有 `static` 方法了，例如 `Stream.of` 就是静态方法：

```Java
public static<T> Stream<T> of(T... values) {
    return Arrays.stream(values);
}
```

### `Optional`

`Stream.reduce` 既可以提供初始值，也可以不提供，不提供初始值时，将直接从 `Stream` 的前两个原始开始计算，此时结果类型为 `Optional`，表示可能不存在有意义的结果。

`orElse` 和 `orElseGet` 用于当 `empty` 时提供默认值，若默认值计算量很大，则使用 `orElseGet`，此时只有是 `empty` 时才会真正开始计算默认值。

## 高级集合类和收集器

### 方法引用

lambda 表达式经常调用参数上的方法：

```Java
artist -> artist.getName()
```

这种做法非常常见，Java 8 为其提供了简写形式，即方法引用：

```Java
Artist::getName
```

构造函数也可用方法引用简化：

```Java
(name, nationality) -> new Artist(name, nationality)
// 简化为
Artist::new
```

* 方法引用 **自动支持多个参数**，只要顺序正确；

可以用方法引用创建数组：

```Java
String[]::new
```

### 元素顺序

在 Java 中有的集合是有序的，例如 `List`，有的集合是无序的，例如 `HashSet`。

`Stream` 按照 **出现顺序** 依次处理流中的元素，出现顺序与 **数据源** 有关，从有序集合创建流，则流是有序的：

```Java
List<Integer> numbers = asList(1, 2, 3, 4);
List<Integer> sameOrder = numbers.stream()
                                 .collect(toList());
assertEquals(numbers, sameOrder);
```

若数据源是 **无序** 的，则生成的流也是无序的：

```Java
Set<Integer> numbers = new HashSet<>(asList(4, 3, 2, 1));
List<Integer> sameOrder = numbers.stream()
                                 .collect(toList());
// 该断言有时会失败
assertEquals(asList(4, 3, 2, 1), sameOrder);
```

流顺序除与数据源有关外，还与流的 **操作** 有关，即使数据源是无序的，但流操作可能产生顺序：

```Java
Set<Integer> numbers = new HashSet<>(asList(4, 3, 2, 1));
List<Integer> sameOrder = numbers.stream()
                                 .sorted()
                                 .collect(toList());
assertEquals(asList(1, 2, 3, 4), sameOrder);
```

例如 `map` 这种中间操作将保持流的顺序：

* 处理的流有序，则 `map` 后的流也有序
* 处理的流无序，则 `map` 后的流也无序

>部分操作在有序流上 **开销更大**，此时可用 `unordered` 取消顺序，但大部分操作在有序流上效率更高，如 `map`、`filter` 和 `reduce` 等。

在并行流上，`forEach` 无法保证按顺序处理，若要保证顺序，需要用 `forEachOrdered`。

### 收集器

`Collector` 是通用的、从流生成复杂值的结构。

JDK 提供的收集器定义在 `java.util.stream.Collectors` 中。

#### 生成其他集合

常见的有 `toList`、`toSet` 和 `toCollection`。

其中 `toList` 和 `toSet` 不需要指定特定的 `List` 或 `Set` 类型，`Stream` 自动挑选合适的具体类型。

若不想底层框架自动挑选，则可用 `toCollection` 来指定具体集合：

```Java
stream.collect(toCollection(TreeSet::new));
```

* `toCollection(f)` 将使用函数 `f` 创建用户想要的集合；

#### 生成值

`maxBy` 和 `minBy` 可按指定顺序生成最大值和最小值：

```Java
// Optional[abc]
Optional<String> m = Stream.of("a", "ab", "abc").collect(Collectors.maxBy(Comparator.comparingInt(String::length)));
```

* 注意找到的是长度最长的字符串，而不是最长的长度；

计算平均值：

```Java
// 2.0
double m = Stream.of("a", "ab", "abc").collect(Collectors.averagingInt(String::length));
```

#### 数据分块

`partitioningBy` 根据 `Predicate` 的结果将流分成两个集合：

```Java
// {false=[1, 3], true=[2, 4]}
Map<Boolean, List<Integer>> partitioned = Stream.of(1, 2, 3, 4).collect(Collectors.partitioningBy(n -> n % 2 == 0));
```

#### 数据分组

`partitioningBy` 只能分成两个集合，`groupingBy` 可将流分成 **任意数量** 的集合：

```Java
// {0=[3, 6], 1=[1, 4], 2=[2, 5]}
Map<Integer, List<Integer>> grouped = Stream.of(1, 2, 3, 4, 5, 6).collect(Collectors.groupingBy(n -> n % 3));
```

* 根据与 3 取模的结果对流分组；
* 类似 SQL 的 group by；

#### 字符串

默认：

```Java
// abc
String result = Stream.of("a", "b", "c").collect(Collectors.joining());
```

指定分隔符：

```Java
// a, b, c
String result = Stream.of("a", "b", "c").collect(Collectors.joining(", "));
```

指定分隔符、前缀、后缀：

```Java
// | a, b, c |
String result = Stream.of("a", "b", "c").collect(Collectors.joining(", ", "| ", " |"));
```

>前两者可用 `String.join` 方法替代，但该方法无法指定前缀和后缀。

#### 组合收集器

计算每个艺术家的专辑数量：

```Java
Map<Artist, List<Album>> albumsByArtist = albums.collect(groupingBy(album -> album.getMainMusician()));

Map<Artist, Integer> numberOfAlbums = new HashMap<>();
for(Entry<Artist, List<Album>> entry : albumsByArtist.entrySet()) {
    numberOfAlbums.put(entry.getKey(), entry.getValue().size());
}
```

以上代码是命令式风格，难以并行化，改写为：

```Java
public Map<Artist, Long> numberOfAlbums(Stream<Album> albums) {
    return albums.collect(groupingBy(album -> album.getMainMusician(), counting()));
}
```

* `groupingBy` 对流分组，对于每个分组，使用 `counting` 进一步处理，结果依然是个 `Map`，不过 value 是 `counting` 处理后的结果；

若要计算每个艺术家的专辑列表，可用 `groupingBy` + `mapping`：

```Java
public Map<Artist, List<String>> nameOfAlbums(Stream<Album> albums) {
    return albums.collect(groupingBy(Album::getMainMusician, mapping(Album::getName, toList())));
}
```

* `mapping` 需要指定结果的集合类型；

#### 自定义收集器

// TODO

#### 通用的 `reduce`

// TODO

收集器模仿了 `reduce` 方法，所有收集器都可以用 `reduce` 实现。

### 一些细节

lambda 表达式推动一些新方法加入集合库，例如 `Map.computeIfAbsent` 等。

之前用 `Map` 实现缓存：

```Java
public Artist getArtist(String name) {
    Artist artist = artistCache.get(name);
    if (artist == null) {
        artist = readArtistFromDB(name);
        artistCache.put(name, artist);
    }
    return artist;
}
```

* 首先尝试从缓存读取值，若结果为 `null`，则从数据库读取，并更新缓存；

可用 `computeIfAbsent` 完美替代：

```Java
public Artist getArtist(String name) {
    return artistCache.computeIfAbsent(name, this::readArtistFromDB);
}
```

真是太方便了！

### 练习

#### 1. 找出名字最长的艺术家

```Java
Stream<String> names = Stream.of("John Lennon", "Paul McCartney", "George Harrison", "Ringo Starr", "Pete Best", "Stuart Sutcliffe");
```

收集器：

```Java
Optional<String> result = names.collect(Collectors.maxBy(Comparator.comparingInt(String::length)));
```

`Stream.max`：

```Java
Optional<String> result = names.max(Comparator.comparingInt(String::length));
```

`Stream.reduce`：

```Java
Optional<String> result = names.reduce((l, r) -> {
    if (l.length() > r.length()) return l;
    else return r;
});
```

#### 2. 每个名字的出现次数

```Java
Stream<String> names = Stream.of("John", "Paul", "George", "John", "Paul", "John");
```

```Java
// {George=1, John=3, Paul=2}
Map<String, Long> result = names.collect(Collectors.groupingBy(name -> name, Collectors.counting()));
```

#### 3. 使用 `computeIfAbsent` 高效的计算斐波那契数列

// TODO

## 数据并行化

从外部迭代到内部迭代，不仅带来更好用的接口，也让并行化变得轻而易举。

### 并行化流操作

* 若已经有一个流，则调用 `parallel`；
* 若要从集合类创建流，则调用 `parallelStream`；

调用 `sequential` 将转换为串行操作，如果同时调用 `sequential` 和 `parallel`，则最后那个生效。

并行流中的 lambda 表达式中不要自行加锁，因为框架会自己处理同步。

>并行流并非速度一定更快，要参考任务规模（太小的话肯定串行更快）、计算资源等。

### 性能

影响并行流速度的主要有 5 个元素：

* 数据大小：只有数据足够大时，并行才有意义；
* 源数据结构
* 装箱：`IntStream` 比 `Stream<Integer>` 快；
* 核的数量
* 元素处理开销：每个元素的处理时间越长，并行带来的好处越大；

并行流基于 fork/join 框架，递归分解问题，并行处理，最后归约合并，因此数据源的数据结构是否容易分治处理，将极大影响并行流的性能：

* 性能好
  + `ArrayList`、`IntStream.range` 和数组，支持随机访问，可轻易分治
* 一般
  + `HashSet` 和 `TreeSet`，可以分治，但不太容易
* 性能差
  + `LinkedList` 难以对半分解
  + `Streams.iterate`、`BufferedReader.lines`，长度未知，难以对半

数据结构对并行流性能影响很大，例如计算 10 000 个数字的和，`ArrayList` 比 `LinkedList` 快 10 倍。

流操作分为有状态和无状态两种：

* 无状态：`map`、`filter`、`flatMap`
* 有状态：`sorted`、`limit`、`distinct`

无状态操作有利于提升性能！

### 并行化数组操作

Java 8 新增了一些数组并行操作函数，都在 `Arrays` 中：

* `parallelPrefix`
* `parallelSetAll`
* `parallelSort`

#### `parallelSetAll`

`parallelSetAll` 并行设置初始值：

```Java
int[] xs = new int[5];
// [0, 1, 2, 3, 4]
Arrays.parallelSetAll(xs, i -> i);
```

* `i` 为数组索引，以上将数组各元素的值初始化为对应索引值；
* 原地修改数组，并未创建新数组！

#### `parallelPrefix`

```Java
public static void parallelPrefix(int[] array, IntBinaryOperator op)
```

更新数组，将每个元素替换为新值，新值由 `IntBinaryOperator` 函数计算而得，以 **当前元素** 以及 **前驱元素** 为参数。

求滑动平均值：

```Java
public static double[] simpleMovingAverage(double[] values, int n) {
    double[] sums = Arrays.copyOf(values, values.length);
    􏰐Arrays.parallelPrefix(sums, Double::sum);
    int start = n - 1;
    return IntStream.range(start, sums.length)
                    .mapToDouble(i -> {
                        double prefix = i == start ? 0 : sums[i - n];
                        return (sums[i] - prefix) / n;
                    })
                    .toArray();􏰔
}
```

## 测试、调试与重构

`peek`：

* 强行打印日志，而不结束流（`map` 之类的方法是 lazy 求值，无法在其中打印日志，`forEach` 可以打印日志，但它是终止方法，导致流无法继续使用，因此使用 `peek`）；
* `peek` 中可以添加断点，此时 `peek` 参数方法可以为空，或是简单的 `i -> i`；
