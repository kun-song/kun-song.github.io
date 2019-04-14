---
title: Kafka Streams 优雅停机
date: 2019-04-06 11:50:49
tags:
  - Kafka
  - 优雅停机
categories: 技术
---

很多框架要求应用（正常 or 异常）退出时，显式进行“善后”处理，如释放资源、防止数据不一致、主动告知自己死亡，避免被动检测带来的时间浪费等等。

下面以 Kafka Streams 为例介绍如何实现优雅停机。

应用关闭后，应显式关闭 Kafka Streams 线程，以防止数据不一致、进行 task 迁移等：

```Java
// KafkaStreams#close()
streams.close();
```

为实现优雅停机，以上代码应该放到 shutdown 钩子中：

```Java
Runtime.getRuntime().addShutdownHook(new Thread(archiveStreams::close));
```

<!-- more -->

shutdown 钩子可以覆盖的场景有：

* Kafka Streams 工作线程出现 **未捕获异常**；
* 外部停止应用（`kill`）
* 应用主动停止（`System.exit(n)`）

但注意一点，在类 UNIX 系统上，几乎任何语言、框架都无法捕获 SIGKILL 信号（`kill -9` 中的 9），当然也包括 JVM：

>In rare circumstances the virtual machine may abort, that is, stop running without shutting down cleanly. This occurs when the virtual machine is terminated externally, for example with the **SIGKILL** signal on Unix or the TerminateProcess call on Microsoft Windows.

>`kill` 不加参数时默认发送 SIGTERM 信号，`kill -l` 即可列出当前系统支持的信号。

因此前面的 Kafka Streams 优雅停机实现有个前提，即关闭脚本未使用 `kill -9`，若非要兼容 `kill -9`，则可：

```Java
final CountDownLatch latch = new CountDownLatch(1);

// attach shutdown handler to catch control-c
Runtime.getRuntime().addShutdownHook(new Thread("streams-shutdown-hook") {
    @Override
    public void run() {
        streams.close();
        latch.countDown();
    }
});

try {
    streams.start();
    latch.await();
} catch (Throwable e) {
    System.exit(1);
}
System.exit(0);
```

以上代码在某工作线程中执行，该线程启动 Kafka Streams 之后即阻塞，若：

* 不用 `kill -9` 停止，则 JVM 将执行 shutdown 钩子，此时线程恢复运行，应用以 `System.exit(0)` 正常退出；
* 用 `kill -9` 停止，则 JVM 不执行 shutdown 钩子，该线程抛出异常，被捕获后，应用以 `System.exit(1)` 异常退出；

以上实现还有一点瑕疵，即 `stream.close()` 未加 **超时参数**，所以即使执行了 shutdown 钩子，应用也可能永远无法退出，这个视实际情况加上即可。

最后，基本所有框架都可按以上套路实现优雅停机，不局限于 Kafka Streams。

---

参考：

* [Kafka Streams shutdown 官方文档](https://kafka.apache.org/documentation/streams/developer-guide/write-streams.html)
* [Kafka Stream: Graceful shutdown](https://stackoverflow.com/questions/50322086/kafka-stream-graceful-shutdown)
* [`Runtime` 官方文档](https://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html)
* [How to gracefully handle the SIGKILL signal in Java](https://stackoverflow.com/questions/2541597/how-to-gracefully-handle-the-sigkill-signal-in-java/2541618#2541618)
* [Dubbo 优雅停机](https://dubbo.incubator.apache.org/zh-cn/docs/user/demos/graceful-shutdown.html)

