---
title: 关于 Java 排序的一切
date: 2018-08-09 21:38:09
tags:
  - Java
  - Comparable
  - Comparator
categories: 技术
---

排序是一项非常基础、常见的功能，几乎所有应用都会涉及，本文介绍 Java 排序的基本知识。

## 基本类型排序

首先排序发生在容器中，在 Java 中表现为数组、`List`、`Set` 和 `Map`。

### 数组排序

>注意 `Arrays.sort` 是就地排序，会修改被排序的数组。

`Arrays.sort` 可以对基本数字类型的数组排序，例如 `int`、`byte`、`char`、`double`、`float`、`short` 和 `long`：

```Java
int[] intArray = new int[]{ 1, 2, 3, -1, -2, -3 };

// -3, -2, -1, 1, 2, 3
Arrays.sort(intArray);
```

<!-- more -->

还可以对数组 **部分元素** 排序：

```Java
// Arrays.sort(int[] a, int fromIndex, int toIndex)
int[] intArray = new int[]{ 1, 2, 3, -1, -2, -3 };

// 1, 2, 3, -2, -1, -3
Arrays.sort(intArray, 3, 5);
```

* 前开后闭区间

若数组很大，可以并发排序（Java 8 引入）：

```Java
int[] intArray = new int[]{ 1, 2, 3, -1, -2, -3 };

// 1, 2, 3, -2, -1, -3
Arrays.parallelSort(intArray, 3, 5);
```

* 除速度可能更快外，`parallelSort` 与 `sort` 其他行为一致；

### `List` 排序

类似数组排序用 `Arrays.sort`，`List` 排序可以用 `Collections.sort`：

```Java
List<Integer> xs = Arrays.asList(1, 2, 3, -1, -2, -3);
// [-3, -2, -1, 1, 2, 3]
Collections.sort(xs);
```

`Collections.sort` 将排序任务委托为 `List.sort`：

```Java
public static <T extends Comparable<? super T>> void sort(List<T> list) {
  list.sort(null);
}
```

`List.sort` 底层通过 `Arrays.sort` 实现：

```Java
default void sort(Comparator<? super E> c) {
  Object[] a = this.toArray();
  Arrays.sort(a, (Comparator) c);
  ListIterator<E> i = this.listIterator();
  for (Object e : a) {
    i.next();
    i.set((E) e);
  }
}
```

### `Set` 和 `Map` 排序

对 `Set` 和 `Map` 中的元素排序，可以把元素先放到 `List` 后，然后对 `List` 排序，最后将元素插入新的 `Set` 和 `Map` 中，注意新 `Set` 和 `Map` 必须用可以保存插入顺序的实现，例如 `LinkedHashSet` 和 `LinkedHashMap`，具体操作可以参考 [这篇博客](https://www.baeldung.com/java-sorting)。

## 自定义对象排序

```Java
class Student {
    private int id;
    private int age;
    private String name;

    public Student(int id, int age, String name) {
        this.id = id;
        this.age = age;
        this.name = name;
    }

    @Override
    public String toString() {
        return "Student{" + "id=" + id + ", age=" + age + ", name='" + name + '\'' + '}';
    }
}

Student mike = new Student(1, 20, "Mike");
Student bob = new Student(2, 18, "Bob");
Student lily = new Student(3, 19, "Lily");

List<Student> studentList = Arrays.asList(mike, bob, lily);
```

与 `int`、`String` 等基本类型不同，自定义对象需要指定排序 **顺序**，有两种指定顺序的方式：

* 自然顺序，通过 `Comparable` 接口指定；
* 任意顺序，通过 `Comparator` 接口指定；

更多关于对象顺序的内容，可以参考 [官方文档](https://docs.oracle.com/javase/tutorial/collections/interfaces/order.html)

### `Comparable`

除基本类型外，可以用 `Collections.sort` 对 `List<String>` 排序，也可以对 `List<Date>` 排序，这是因为 `String` 实现了 `Comparable` 接口，从而为 `String` 提供了自然顺序，Java 中很多类型实现了 `Comparable` 接口，它们都可以用 `sort` 排序。

如果想用 `Collections.sort` 对自定义对象的 `List` 排序，则该类必须实现 `Comparable` 接口，该接口非常简单：

```Java
public interface Comparable<T> {
    public int compareTo(T o);
}
```

`compareTo` 返回值表示本对象与 `o` 的大小关系：

* 正数：本对象大于 `o`
* 0：本对象等于 `o`
* 负数：本对象小于 `o`

让 `Student` 实现 `Comparable` 接口：

```Java
class Student implements Comparable<Student> {
    ...
    @Override
    public int compareTo(Student o) {
        return o.id - id;
    }
}
```

这里按 `id` 逆序排序：

```Java
// [Student{id=3, age=19, name='Lily'}, Student{id=2, age=18, name='Bob'}, Student{id=1, age=20, name='Mike'}]
Collections.sort(studentList);
```

还可以按 `name` 字段排序，重新将 `compareTo` 实现为：

```Java
public int compareTo(Student o) {
  return this.name.compareTo(o.name);
}
```

排序结果：

```Java
// [Student{id=2, age=18, name='Bob'}, Student{id=3, age=19, name='Lily'}, Student{id=1, age=20, name='Mike'}]
Collections.sort(studentList);
```

还可以先按 `age` 从小到大排序，若 `age` 相同，再按 `name` 排序：

```Java
public int compareTo(Student o) {
  int cmp = age - o.age;
  return cmp == 0 ? name.compareTo(o.name) : cmp;
}
```

排序结果：

```Java
// [Student{id=2, age=18, name='Bob'}, Student{id=2, age=18, name='Bob2'}, Student{id=3, age=19, name='Lily'}, Student{id=1, age=20, name='Mike'}]
Collections.sort(studentList);
```

### `Comparator`

若不想用自然顺序排序，或排序对象未实现 `Comparable` 接口，则可用 `Comparator` 接口实现排序，`Comparator` 实例封装了某种顺序：

```Java
public interface Comparator<T> {
    int compare(T o1, T o2);
}
```

* `compare` 方法语义与 `compareTo` 基本相同，同样通过正数、0、负数表示顺序。

前面提到的 `Arrays.sort`、`Collections.sort` 和 `List.sort` 都可以指定一个 `Comparator` 参数，以自定义顺序。

用 `age` 排序：

```Java
// [Student{id=2, age=18, name='Bob'}, Student{id=3, age=19, name='Lily'}, Student{id=1, age=20, name='Mike'}]
Collections.sort(studentList, (a, b) -> a.getAge() - b.getAge());
```

Java 8 为 `Comparator` 新增 `comparingInt` 方法，上面代码可改写为：

```Java
Collections.sort(studentList, Comparator.comparingInt(Student::getAge));
```

Java 8 新增 `thenComparing` 方法，用于串联任意多比较元素，例如 先按 `age` 后按 `name` 排序：

```Java
Collections.sort(studentList, Comparator.comparingInt(Student::getAge).thenComparing(Student::getName));
```

* `thenComparing` 让代码意图非常清晰。

### `Comparable` 和 `Comparator` 必须遵循的约束

// TODO

The compare method has to obey the same four technical restrictions as Comparable's compareTo method for the same reason — a Comparator must induce a total order on the objects it compares.

