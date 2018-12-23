---
title: JVM | jps
date: 2018-12-23 20:39:39
tags:
  - Java
  - JVM
  - jps
categories: JVM
---

jps（Java Virtual Machine Process Status Tool）类似 Unix 下的 ps（Process Status Tool），不过只能查看 Java 进程的状态。

默认情况下，jps 仅输出 Java 进程的 LVMID 和主类名字：

```
± jps
14528 Server
13938 sbt-launch.jar
17881 Jps
```

<!-- more -->

> LVMID 即本地虚拟机唯一 ID（Local Virtual Machine Identifier），对于本地 Java 进程而言，LVMID 与本地操作系统的进程 ID（可通过 ps 获取）相同。

若只关心 LVMID，可通过 `-q` 去掉主类名字：

```
± jps -q
14528
13938
18364
```

若本地启动了同一 app 的多个实例，可通过 `-l` 显式主类的全名，若是 jar 包，则显式 jar 包的路径：

```
± jps -l
14528 org.ensime.server.Server
13938 /usr/local/Cellar/sbt/1.2.7/libexec/bin/sbt-launch.jar
18571 sun.tools.jps.Jps
```

可通过 `-m` 查看启动时传递给 main 函数的参数：

```
± jps -m
14528 Server
13938 sbt-launch.jar
18654 Jps -m
```

最后，可通过 `-v` 查看启动时通过命令行传递的 JVM 参数：

```
± jps -v
14528 Server -Dscala.classpath.closeZip=true -XX:ReservedCodeCacheSize=128m -XX:MaxMetaspaceSize=256m -Xss2m -Xms512m -Xmx4g -XX:MaxMetaspaceSize=256m -XX:StringTableSize=1000003 -XX:+UnlockExperimentalVMOptions -XX:SymbolTableSize=1000003 -Densime.index.no.reverse.lookups=true -Densime.config=/Users/satansk/Projects/jvm-scala/.ensime
18802 Jps -Dapplication.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_66.jdk/Contents/Home -Xms8m
13938 sbt-launch.jar -Xms1024m -Xmx1024m -XX:ReservedCodeCacheSize=128m -XX:MaxMetaspaceSize=256m
```

与 jinfo 不同的是，jps 只能查看显式传递的参数，无法查看系统默认参数。
