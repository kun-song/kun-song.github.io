---
title: 类加载（二）：何时加载类？
date: 2019-11-23 15:47:49
tags:
  - 类加载
categories: JVM
---

类加载机制的第一步是“加载”，但JVM 规范 **没有强制** 规定何时执行加载，由 JVM 实现自行处理；但 JVM 规范规定了类“初始化”的时机，因此类加载的时机实际由类初始化 **间接决定**。

<!-- more -->

## 主动引用：触发初始化

但对于“初始化”这步，JVM 规范严格规定了 5 种场景必须 **立即** 对类进行“初始化”，前面的加载、验证、准备自然也必须执行：

1. JVM 启动时，必须立即初始化用户指定的 **主类**（`main` 方法所在的类）；
2. JVM 执行时，遇到 `new`、`getstatic`、`putstatic`、`invokestatic` 4 条指令时，必须立即初始化对应类：
   1. 使用 `new` 创建实例时，初始化目标类；
   2. 读取、设置类的 **静态字段** 时，初始化 **该字段所在的类**；
   3. 调用类的 **静态方法** 时，初始化 **该方法所在的类**；
3. 子类的初始化会触发其父类的初始化；
4. 若接口定义了默认方法（`default` 关键字），则（直接 or 间接）实现该接口的类的初始化，将触发该接口的初始化；
5. 使用 **反射** API 对类进行反射调用时，初始化该类；
6. 当初次调用 MethodHandle 实例时，初始化该 MethodHandle 指向的方法所在的类；

以上对类的引用方式称为 **主动引用**，其它方式称为 **被动引用**，任何被动引用都 **不会** 触发类的初始化。

## 被动引用：不会触发初始化

通过 `-XX:+TraceClassLoading` 可以打印 HotSpot 类加载的详细信息。

### 通过子类访问父类的静态字段、方法

JVM 遇到访问静态字段、静态方法的字节码指令（`getstatic`、`putstatic`、`invokestatic`）时，会初始化该字段、方法 **所在** 的类，这里的“所在”指 **直接定义** 该字段的类。

注意，通过子类访问父类的静态字段、方法时，仅会初始化 **父类**，**不会** 初始化子类：

```Java
public class Super {
    public static int index = 666;

    static {
        System.out.println("Super init.");
    }
}

public class Sub extends Super {
    static {
        System.out.println("Sub init.");
    }
}
```

以上类定义中的 `static` 块将被 Javac 编译到 `<clinit>` 方法中，该方法在“初始化”阶段执行。

执行 `System.out.println(Sub.index)` 时，输出：

```Java
...
[Loaded Super from file:/Users/satansk/Projects/github/akka/akkastreamsexpectNextN/target/classes/]
[Loaded Sub from file:/Users/satansk/Projects/github/akka/akkastreamsexpectNextN/target/classes/]
Super init.
666
```

可见未执行子类 `Sub` 的 `<clinit>` 方法，即未执行其初始化。

值得注意的是，JVM 规范 **并未阻止** JVM 实现执行子类的类加载，不同虚拟机实现也可以加载子类，从上面输出可以看到，HotSpot 触发了 `Sub` 的加载，但没有初始化它。

### 通过数组定义引用类

```Java
Super[] xs = new Super[0];
```

以上不会触发 `Super` 类的初始化，但触发了 `[Super` 类的初始化，该类是 JVM 自动生成的，代表数组类型。

### 访问常量不会触发类的初始化

这里的常量指用 `static final` 修饰的字段，前面说过访问 **静态字段** 会触发所在类的初始化，但添加 `final` 后一切变的不同：

```Java
public class Super {
    public static final int index = 666;

    static {
        System.out.println("Super init.");
    }
}
```

执行 `System.out.println(Sub.index)` 仅输出 666，说明没有执行 `Super` 类的初始化。

原因是编译时通过常量传播优化，`index` 字段已经被移动到使用类（即 `System.out.println(Sub.index)` 所在类） 的 **常量池** 中，编译成 class 文件后，该类与 `Super` **没有任何关系**，也无法引用到 `Super` 了，自然也不会触发其初始化。
