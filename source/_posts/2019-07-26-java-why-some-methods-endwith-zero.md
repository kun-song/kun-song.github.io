---
title: 为什么有的 Java 方法以 0 结尾？
date: 2019-07-26 00:15:39
tags:
categories: Java
---

在阅读 JDK 或框架源码的时候，常常发现部分方法是以 0 结尾的，凭直觉这并非良好的命名方式，但的确广泛存在，这是为什么呢？

<!-- more -->

参考网络上的一些资料，个人认为这种命名方式最初是为了关联 native 方法与其对应的包装器方法，例如 `Thread`：

```Java
public class Thread implements Runnable {
    public final void setPriority(int newPriority) {
        ThreadGroup g;
        checkAccess();
        if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
            throw new IllegalArgumentException();
        }
        if((g = getThreadGroup()) != null) {
            if (newPriority > g.getMaxPriority()) {
                newPriority = g.getMaxPriority();
            }
            setPriority0(priority = newPriority);
        }
    }

    /* Some private helper methods */
    private native void setPriority0(int newPriority);
    private native void stop0(Object o);
    private native void suspend0();
    private native void resume0();
    private native void interrupt0();
    private native void setNativeName(String name);
}
```

真正设置线程优先级的是 native 的 `setPriority0` 方法，但 Java 为了提升安全性，没有直接暴露该 native 方法，而是在其上包装了一下，添加权限控制、优先级数值校验等功能，验证合法之后，才调用 `setPriority0` 去真正的设置线程的优先级。

这两个方法的关联极强，一个是 public，一个是 private，因此将 private native 方法以 0 结尾，表示它是一个辅助方法。

后来起辅助作用的非 native 但 private 的方法也开始以 0 结尾，这时 0 用来区分同名的 public/private 两个方法，例如 `Class`：

```Java
public final class Class<T> implements ... {
    public Field getField(String name) throws NoSuchFieldException, SecurityException {
        Objects.requireNonNull(name);
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkMemberAccess(sm, Member.PUBLIC, Reflection.getCallerClass(), true);
        }
        Field field = getField0(name);
        if (field == null) {
            throw new NoSuchFieldException(name);
        }
        return getReflectionFactory().copyField(field);
    }

    private Field getField0(String name) {
        Field res;
        if ((res = searchFields(privateGetDeclaredFields(true), name)) != null) {
            return res;
        }
        Class<?>[] interfaces = getInterfaces(/* cloneArray */ false);
        for (Class<?> c : interfaces) {
            if ((res = c.getField0(name)) != null) {
                return res;
            }
        }
        if (!isInterface()) {
            Class<?> c = getSuperclass();
            if (c != null) {
                if ((res = c.getField0(name)) != null) {
                    return res;
                }
            }
        }
        return null;
    }
}
```

虽然 `getField0` 不是 native 方法，但还是 private 方法，与前面类似，包含获取字段的真正逻辑，而 `getField` 是对外暴露的 public 方法，仅在 `getField0` 基础上做一些安全检查。

这种命名在 Java 生态中非常常见，不止 JDK，其他框架如 Netty 中也有类似做法。

---

参考：

1. [Why some java methods in core libraries end with numbers?](https://stackoverflow.com/questions/10006293/why-some-java-methods-in-core-libraries-end-with-numbers)
