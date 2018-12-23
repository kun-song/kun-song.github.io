---
title: JVM | jinfo
date: 2018-12-23 19:35:30
tags:
  - Java
  - JVM
  - jinfo
categories: JVM
---

jinfo 是 JVM 配置信息的查看、修改工具，man 手册介绍如下：

>jinfo prints Java configuration information for a given Java process or core file or a  remote  debug server. Configuration  information includes Java System properties and Java virtual machine command line flags.

且：

>This utility is unsupported and may or may not be available in future  versions  of  the  J2SE SDK.

目前 Java 11 还是支持 jinfo 的，不过 jinfo 的 JDK 版本要与 app 一致才行。

<!-- more -->

配置信息分为两种：

* command line flags -- JVM 启动时通过命令行显式传递的参数；
* System properties -- 未显式指定的 **系统默认值**；

不指定选项时，默认两种信息都打印，若只看其中一种，可通过：

* `-flags` -- 打印命令行配置
* `-sysprops` -- 打印系统配置

例如：

```
± jinfo 13938
Attaching to process ID 13938, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.66-b17

Java System Properties:

jline.esc.timeout = 0
java.runtime.name = Java(TM) SE Runtime Environment
jna.platform.library.path = /usr/lib:/usr/lib
java.vm.version = 25.66-b17
...


VM Flags:

Non-default VM flags: -XX:CICompilerCount=3 -XX:InitialHeapSize=1073741824 -XX:MaxHeapSize=1073741824 -XX:MaxMetaspaceSize=268435456 -XX:MaxNewSize=357564416 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=357564416 -XX:OldSize=716177408 -XX:ReservedCodeCacheSize=134217728 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC 
Command line:  -Xms1024m -Xmx1024m -XX:ReservedCodeCacheSize=128m -XX:MaxMetaspaceSize=256m
```

* 配置信息默认分为 `Java System Properties` 和 `VM Flags` 两类。

只看 `VM Flags`：

```
± jinfo -flags 13938
Attaching to process ID 13938, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.66-b17
Non-default VM flags: -XX:CICompilerCount=3 -XX:InitialHeapSize=1073741824 -XX:MaxHeapSize=1073741824 -XX:MaxMetaspaceSize=268435456 -XX:MaxNewSize=357564416 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=357564416 -XX:OldSize=716177408 -XX:ReservedCodeCacheSize=134217728 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC 
Command line:  -Xms1024m -Xmx1024m -XX:ReservedCodeCacheSize=128m -XX:MaxMetaspaceSize=256m
```

只看 `Java System Properties`：

```
± jinfo -sysprops 13938
Attaching to process ID 13938, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.66-b17
jline.esc.timeout = 0
java.runtime.name = Java(TM) SE Runtime Environment
...
```

只查看指定参数的值：

```
± jinfo -flag SurvivorRatio 13938
-XX:SurvivorRatio=8
```

修改 `SurvivorRatio` 的值：

```
± jinfo -flag SurvivorRatio=9 13938
```

结果报错了：

```
Exception in thread "main" com.sun.tools.attach.AttachOperationFailedException: flag 'SurvivorRatio' cannot be changed

	at sun.tools.attach.BsdVirtualMachine.execute(BsdVirtualMachine.java:213)
	at sun.tools.attach.HotSpotVirtualMachine.executeCommand(HotSpotVirtualMachine.java:261)
	at sun.tools.attach.HotSpotVirtualMachine.setFlag(HotSpotVirtualMachine.java:234)
	at sun.tools.jinfo.JInfo.flag(JInfo.java:134)
	at sun.tools.jinfo.JInfo.main(JInfo.java:81)
```

原因是 JVM 启动默认开启 `-XX:+UseAdaptiveSizePolicy`，它与 jinfo 设置参数冲突，参考 [stackoverflow 该回答](https://stackoverflow.com/questions/42719887/survivorratio-flag-of-jvm-does-not-work)。

换一个参数：

```
± jinfo -flag HeapDumpPath 13938              
-XX:HeapDumpPath=
```

没有设置该值，用 jinfo 设置一下：

```
± jinfo -flag HeapDumpPath=/home/satansk/xx.hprof 13938
```

再看下 `HeapDumppath` 的值;

```
± jinfo -flag HeapDumpPath 13938                       
-XX:HeapDumpPath=/home/satansk/xx.hprof
```

nice ~~

