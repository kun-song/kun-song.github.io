---
title: 类加载（六）：双亲委派模型
date: 2019-11-24 14:15:42
tags:
  - 类加载
categories: JVM
---

HotSpot 虚拟机默认提供 3 个类加载器：

* 启动类加载器（Bootstrap ClassLoader）：HotSpot 内置，属于虚拟机的一部分，C++ 实现；
* 扩展类加载器（Extension ClassLoader）：JDK 内置，Java 实现；
* 应用类加载器（Application ClassLoader）：JDK 内置，Java 实现；

<!-- more -->

除启动类加载器外，其他所有类加载器（包括用户自定义的类加载器）都是 `java.lang.ClassLoader` 的子类，都由 Java 实现。

>应用类加载器是 `ClassLoader.getSystemClassLoader()` 方法的返回值，所以又被称为 **系统类加载器**。

## 默认类加载器

`Class.getClassLoader()` 方法可获取加载目标类的类加载器。 

### 启动类加载器

所有 Java 类都由类加载器加载，类加载器本身也是 Java 类，所以需要一个“独特”的类加载器迈出第一步，即启动类加载器。

它负责加载 JDK 核心类库，即以下路径中的类：

* `$JAVA_HOME/jre/lib`（主要是 `rt.jar`）
* `-Xbootclasspath` 参数指定的路径

>仅加载目标路径中，可被识别的类文件，若文件无法被 HotSpot 识别，直接忽略。

集合类包含在 `rt.jar` 文件中，因此 `ArrayList` 由启动类加载器加载：

```Java
System.out.println(ArrayList.class.getClassLoader());
null  // 输出
```

可见 `getClassLoader` 返回 `null` 即表示该类为启动类加载器加载。

>启动类加载器属于虚拟机 **实现** 的一部分，不属于 JDK 类库，因此不同 JVM 实现中的启动类加载器并不相同。

### 扩展类加载器

加载 Java 核心类库的扩展，即以下路径中的类：

* `$JAVA_HOME/lib/ext`
* `java.ext.dirs` 系统变量指定的路径

```Java
System.out.println(Logging.class.getClassLoader());
sun.misc.Launcher$ExtClassLoader@2c7b84de  // 输出
```

### 系统类加载器

加载应用自己定义的类，即：

* `classpath` 环境变量指定的路径
* `-classpath`/`-cp` 参数指定的路径
* `java.class.path` 系统变量指定的路径

```Java
System.out.println(Logging.class.getClassLoader());
sun.misc.Launcher$AppClassLoader@18b4aac2  // 输出
```

## 双亲委派模型

### 内容

类加载器之间具备 **层次关系**，除启动类加载器外，其余每个类加载器都有一个 **父类加载器**，这种父子关系通过 **组合** 而非 **继承** 实现。

双亲委派模型下，类的加载过程如下：

* **向上** 委托加载请求：类加载器接收到类加载请求后，不会自己尝试加载，而是将加载请求向上委托给自己的 **父类加载器**，最终 **所有** 加载请求都会到达启动类加载器；
* **向下** 反馈加载结果：父类加载器在自己的 **搜索范围** 内搜索目标类：
  * 发现：加载该类，并返回给子类加载器；
  * 未发现：向子类加载器反馈自己无法加载该类；

只有当父类加载器反馈自己 **无法加载** 时，子类加载器才会尝试自己加载，若子类加载器也无法加载，则抛出异常。

### 价值

JVM 中类的唯一性由两点决定：

* 类本身
* 该类的类加载器

因此若无双亲委派模型，Java **核心类库** 可能被不同类加载器加载，它们会被认为是不同的类，因此不同应用中 `ArrayList` 可能被认为是完全不同的多个类，造成很大混乱。

通过双亲委派模型，Java 类之间形成了一种 **层次关系**，类似 `ArrayList` 的核心类无论最初被什么加载器加载，**最终** 都会被启动类加载器加载。

### 实现

自定义一个 **遵守双亲委派模型** 的类加载器很简单，只需：

1. 继承 `ClassLoader` 抽象类；
2. 实现 `findClass(String name)`； 

`findClass` 注释如下：

>This method should be overridden by class loader implementations that follow the **delegation model** for loading classes, and will be invoked by `loadClass` method after checking the parent class loader for the requested class.  The default implementation throws a `ClassNotFoundException`.

`ClassLoader` 内置了 **双亲委派的实现逻辑**，该逻辑在 `loadClass` 方法中：

```Java
// First, check if the class has already been loaded
Class<?> c = findLoadedClass(name);
if (c == null) {
    // 向上委托加载请求，让父类加载器 or 启动类加载器加载
    try {
        if (parent != null) {
            c = parent.loadClass(name, false);
        } else {
            c = findBootstrapClassOrNull(name);
        }
    } catch (ClassNotFoundException e) {
        // ClassNotFoundException thrown if class not found
        // from the non-null parent class loader
    }
    
    // 父类加载器无法加载，则使用当前加载器的 findClass 加载
    if (c == null) {
        c = findClass(name);
    }
}
```

`parent` 字段代表的类加载器，前面说过启动类加载器是 C++ 实现，JDK 中用 `null` 表示，因此若 `parent` 为 `null`，则表示当前类加载器的父类加载器就是启动类加载器。

### 违反

双亲委派模型在 Java 1.2 引入，是官方推荐的类加载实现方式，并非一种强制约束，有充足理由时，也可以违反：

* 基础类调用用户代码：SPI 机制
* 动态性：OSGI
