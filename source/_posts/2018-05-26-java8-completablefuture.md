---
title: 【译】CompletableFuture 指南
date: 2018-05-27 11:42:38
tags:
  - Java 8
  - CompletableFuture
categories: Java
---

本文翻译自 [Guide To CompletableFuture](http://www.baeldung.com/java-completablefuture)。

Java 5 之前很难处理 **异步计算**，那时一般用回调函数处理异步计算的结果，业务逐渐复杂后，会遇到臭名昭著的 *回调地狱*。

Java 5 引入 `Future` 接口，用来代表异步计算的结果，摆脱了回调函数，但 `Future` 并没有提供任何 **组合** `Future` 的方法，也无法处理异步计算中可能出现的错误。

Java 8 引入了大杀器 `CompletableFuture`，该类实现了 `Future` 接口和 `CompletionStage` 接口，`CompletionStage` 接口代表异步计算中的 **不同阶段**，以及如何 **组合** 这些计算阶段。

<!-- more -->

`CompletableStage` 接口中有 50 多个方法，可以对 `CompletableStage` 进行组合、计算，方法看似很多，但可以按功能对其分类，大多数方法都有 3 种变体：

* 不带 `Async` 版本：同步方法
* 带 `Async`，只有一个参数：异步方法，使用默认的 `ForkJoinPool.commonPool()` 获取线程池，并使用
* 带 `Async`，有两个参数：异步方法，且使用第二个参数指定的 `ExecutorService` 线程池

## 用作普通 `Future`

### 完成 `Future`

`CompletableFuture` 实现了 `Future` 接口，可以将其视为带有 `complete` 方法的 `Future`。

例如，我们可以创建一个 `CompletableFuture` 实例，表示未来某个时间才能获取到的计算结果，并将其传递给消费者，消费者在任务完成后，调用 `CompletableFuture.complete` 方法完成任务：

```Java
public class CompletableFutureDemo {

    public static Future<String> calculateAsync() {
        // 异步计算结果
        CompletableFuture<String> future = new CompletableFuture<>();

        Executors.newCachedThreadPool().submit(() -> {
            // 模拟业务处理
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            // 完成任务，将结果写入 CompletableFuture
            future.complete("hello");
        });

        return future;
    }
}
```
* 虽然 `calculateAsync` 方法内部使用的是 `CompletableFuture`，但其返回值类型为 `Future`
* `calculateAsync` 方法内部利用了 `Completable.complete` 方法，但外部仅暴露 `Future` 相关方法
* `calculateAsync` 方法为异步方法，会立刻返回 `Future`，但是 `Future` 中真正的结果需要等任务实际完成后才能获取

当 `caculateAsync` 的调用者若要获取其计算结果，需要做好 **阻塞** 的准备，因为 `Future` 暴露的方法无非阻塞版本：

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Future<String> f = calculateAsync();

        // 准备好阻塞线程，并处理异步计算结果
        String result = f.get();

        // 5s 后，输出 hello
        System.out.println(result);
    }
}
```

若异步计算结果已经明确，不需要计算，则可以直接使用 `CompletableFuture.completedFuture` 方法创建包含该值的 `CompletableFuture`，且此时 `Future.get()` 将不再阻塞，会马上返回：

```Java
public class CompletableFutureDemo {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // 结果已知，直接使用该值创建 CompletableFuture
        Future<String> f = CompletableFuture.completedFuture("hello");

        // 此时 Future.get 不再阻塞
        String result = f.get();

        // 马上输出
        System.out.println(result);
    }

}
```

### 取消 `Future`

若根本无法得到计算结果，则取消异步计算是个不错的想法：

```Java
public class CompletableFutureDemo {

    public static Future<String> calculateAsync() {
        // 异步计算结果
        CompletableFuture<String> future = new CompletableFuture<>();

        Executors.newCachedThreadPool().submit(() -> {
            // 模拟业务处理
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            // 经过努力，无法完成任务，则需要取消 Future
            // Future.cancel(flag) 中的 flat 标志为对 CompletableFuture 毫无影响
            future.cancel(false);
        });

        return future;
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Future<String> f = calculateAsync();

        // 若 Future 被取消，则 Future.get 会抛出 CancellationException
        String result = f.get();

        System.out.println(result);
    }

}
```

注意，若调用 `Future.get` 时，`Future` 已经被取消，则该方法抛出 `CancellationException` 异常。

## 封装计算

从前面的例子可以看到，`CompletableFuture` 可以与现有的并发框架完美适配，但有时，我们想抛弃那些古老的并发有关的样板代码（比如创建线程池之类），直接异步地执行某些代码，这当然可以！

静态方法 `runAsync` 和 `supplyAsync` 可以使用 `Runnable` 和 `Supplier` 类型创建 `CompletableFuture` 实例，而且 `Runnable` 和 `Supplier` 都是函数式的接口，它们的实例可以被当做 lambda 表达式使用。

* `Runnable` 就是那个古老的 `Runnable` 接口，不能包含返回值
* `Supplier` 是个新接口，包含一个不接受入参，且返回值类型可配置的函数

因为 `Supplier` 实例为 lambda 表达式，所以可以将 lambda 表达式作为参数，调用 `supplyAsync` 函数：

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // supplyAsync 接受 supplier 实例作为参数
        CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> "hello");

        String result = f.get();

        System.out.println(result);
    }

}
```

## 处理异步计算的结果

>thenXXX 都是阻塞方法，需要等待前面的 `Future` 执行完成。

### 接受上次的处理值，返回再次处理的值

异步计算完成后，一般会把结果传入某个函数继续处理，`thenApply` 方法正是为此而生：

1. 接受一个函数参数
2. 使用该函数处理 `CompletableFuture` 的计算结果，并将其计算结果包裹为新的 `CompletableFuture` 返回

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // supplyAsync 接受 supplier 实例作为参数
        CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> "hello");

        // thenApply 处理 f 的计算结果
        CompletableFuture<String> f2 = f.thenApply(x -> x + " world");

        String result = f2.get();

        // 输出 hello world
        System.out.println(result);
    }

}
```

### 接受上次处理的值，返回 `Void`

若不想将计算在 `Future` 链中传递下去，则可选择 `thenAccept` 取代 `thenApply`：

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // supplyAsync 接受 supplier 实例作为参数
        CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> "hello");

        // thenAccept 处理 f 的计算结果，但返回 Void
        CompletableFuture<Void> f2 = f.thenAccept(x -> System.out.println("CompletableFuture stops here, result = " + x));

        // Future.get 将返回 Void
        Void result = f2.get();

        // 输出 null
        System.out.println(result);
    }

}
```
* `CompletableFuture.thenAccept` 返回的 `Future` 上调用 `get` 将返回 `Void`

### 不接受上次处理的值，返回 `Void`

该场景使用 `thenRun` 方法：

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // supplyAsync 接受 supplier 实例作为参数
        CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> "hello");

        // thenAccept 处理 f 的计算结果，但返回 Void
        CompletableFuture<Void> f2 = f.thenRun(() -> System.out.println("CompletableFuture stops here"));

        // Future.get 返回 Void，并产生副作用：终端输出 CompletableFuture stops here
        Void result = f2.get();
    }

}
```
* `thenRun` 接受 `Runnable` 参数，唯一作用就是产生副作用

## 组合 `Future`

### 前后有依赖关系

`CompletableFuture` 最奇妙的是其可组合性，实际上 `CompletableFuture` 就是一个 `Monad` 实例，而 `thenCompose` 对应于 `flatMap`，这完全就是函数式的一套东西。

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // thenCompose 用于组合两个 CompletableFuture，返回值也是 CompletableFuture，可用于进一步组合
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "hello")
                .thenCompose(r -> CompletableFuture.supplyAsync(() -> r + " world"));

        String result = f2.get();

        // hello world
        System.out.println(result);
    }

}
```

注意 `thenApply` 与 `thenCompose` 是 `CompletableFuture` Monad 的核心 API，对应 Scala 中的 `map` 和 `flatMap`。

### 独立 `Future`

如果要运行两个独立的 `Future`，且需要处理两者的计算结果，则使用 `thenCombine`：

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // thenCombine 用于组合两个独立的 CompletableFuture
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "hello")
                .thenCombine(CompletableFuture.supplyAsync(() -> " world"), (a, b) -> a + b);

        String result = f2.get();

        // hello world
        System.out.println(result);
    }

}
```

### 独立 `Future`，无返回值

`thenCompose` 和 `thenCombine` 都返回计算结果，如果不需要结果，则使用 `thenAcceptBoth`：

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // thenAcceptBoth 用于组合两个独立的 CompletableFuture，无返回值，使用其副作用
        CompletableFuture<Void> f2 = CompletableFuture.supplyAsync(() -> "hello")
                .thenAcceptBoth(CompletableFuture.supplyAsync(() -> " world"), (a, b) -> System.out.println(a + b));

        // get 副作用为输出 hello world
        Void result = f2.get();

        // null
        System.out.println(result);
    }

}
```

## 并行处理 `Future`

当并行处理多个 `Future` 时，一般需要等待所有 `Future` 执行结束，将它们的结果作为整体进行处理。

`CompletableFuture.allOf` 静态方法就是为此而生，它会等待所有参数 `Future`，直到它们都完成：

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "hello");
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> " beautiful");
        CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> "world");

        // allOf 等待所有 Future 执行结束，其返回值为 Void
        CompletableFuture<Void> f = CompletableFuture.allOf(f1, f2, f3);

        Void result = f.get();

        System.out.println(f1.isDone());  // true
        System.out.println(f2.isDone());  // true
        System.out.println(f3.isDone());  // true
    }

}
```

* `allOf` 返回值为 `Void`，并不能反映参数 `Future` 的执行结果
* 各个 `Future` 的具体结果，需要各个查询

幸运的是，使用 `CompletableFuture` 和 `Stream` 可以轻易获取各个 `Future` 的计算结果：

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "hello");
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "beautiful");
        CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> "world");

        // allOf 等待所有 Future 执行结束，其返回值为 Void
        CompletableFuture<Void> f = CompletableFuture.allOf(f1, f2, f3);

        Void result = f.get();

        System.out.println(f1.isDone());  // true
        System.out.println(f2.isDone());  // true
        System.out.println(f3.isDone());  // true

        String re = Stream.of(f1, f2, f3)
                .map(CompletableFuture::join)
                .collect(Collectors.joining(" "));

        // hello beautiful world
        System.out.println(re);
    }

}
```
* `CompletableFuture.join` 类似 `Future.get` 方法，区别是，`join` 在无法正常完成任务时，抛出 unchecked exception，而 `get` 抛出 checked exception，从而使的 `join` 可以用于 `Stream`

## 错误处理

### `exceptionally`

```Scala
/**
  * Returns a new CompletableFuture that is completed when this
  * CompletableFuture completes, with the result of the given
  * function of the exception triggering this CompletableFuture's
  * completion when it completes exceptionally; otherwise, if this
  * CompletableFuture completes normally, then the returned
  * CompletableFuture also completes normally with the same value.
  * Note: More flexible versions of this functionality are
  * available using methods {@code whenComplete} and {@code handle}.
  *
  * @param fn the function to use to compute the value of the
  * returned CompletableFuture if this CompletableFuture completed
  * exceptionally
  * @return the new CompletableFuture
  */
public CompletableFuture<T> exceptionally(
  Function<Throwable, ? extends T> fn) {
  return uniExceptionallyStage(fn);
}
```

如下，`f` 函数可能抛出异常，但此处没有使用 `Try`，使用 `exceptionally` 方式处理可能出现的异常：

```Scala
CompletableFuture<File> f2 = CompletableFuture.supplyAsync(() -> f(true));

f2.exceptionally(e -> {
  System.out.println("f2 exception = " + e);
  return null;
});
```
* `f` 若抛出异常，则会调用 `exceptionally` 中指定的回调函数，该函数处理异常，并返回一个异常时的 **默认值**，以供以后的链式调用使用
* 此处默认值为 `null`

## `handle`

`CompletableFuture` 链中，异常处理与 `try-catch` 略有不同，`CompletableFuture` 的错误是在 `handle` 方法中处理的，该方法接受一个函数作为参数，该函数的入参为 `(result, Exception)` 对，第一个是异步计算的结果（如果正常结束），第二个是异步结算抛出的异常（如果非正常结束）。

```Java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        String name = null;

        CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> {
            if (name == null)
                throw new RuntimeException("name can't be null");
            return "hello " + name;
        }).handle((result, exception) -> result == null ? "hello stranger!" : result);

        // hello stranger!
        String result = f.get();

        System.out.println(result);
    }

}
```

### `completedException`

## `get` 与 `join` 的区别

`get` 是 `Future` 定义的方法，`join` 是 `CompletableFuture` 新添加的方法，两者都可以从 `CompletableFuture` 中提取计算结果，它们 **唯一** 的区别是如何抛出异常。

### `get` 抛出 checked exception

`get` 方法签名如下：

```Java
public T get() throws InterruptedException, ExecutionException
```

它抛出的是 checked exception，必须用 `try-catch` 包裹。

### `join` 抛出 unchecked exception

`join` 签名如下：

```Java
/**
  * Returns the result value when complete, or throws an
  * (unchecked) exception if completed exceptionally. To better
  * conform with the use of common functional forms, if a
  * computation involved in the completion of this
  * CompletableFuture threw an exception, this method throws an
  * (unchecked) {@link CompletionException} with the underlying
  * exception as its cause.
  *
  * @return the result value
  * @throws CancellationException if the computation was cancelled
  * @throws CompletionException if this future completed
  * exceptionally or a completion computation threw an exception
  */
public T join()
```

可见，`join` 签名中未声明要抛出的异常，而是在注释中说明该方法在被取消，或者异常完成时，会分别抛出 `CancellationException` 和 `CompletionException`，它们都是 unchecked exception，不需要用 `try-catch` 包裹。
