---
title: 使用 jenv 进行 JDK 版本管理
date: 2018-12-16 12:37:48
tags:
  - Java
  - jenv
categories: 编程工具
---

很多框架（比如 Dubbo）需要兼容多个 JDK 版本，开发、测试时需要不断切换版本，通过 `JAVA_HOME` 手动修改让人痛苦不堪。

据我了解 Node.js 有很多版本管理工具，例如 nvm，通过 nvm 切换 Node.js 版本只需要一个命令，非常方便高效，幸运的是 Java 也有类似工具，即 jenv。

因为我的 mac os 已经装了 Java 11，所以直接安装 jenv：

```
brew install jenv
```

<!-- more -->

安装完成后，执行以下脚本：

```
echo 'eval "$(jenv init -)"' >> ~/.zshrc
```

因为我用的是 zsh，所以初始化写到 `.zshrc` 里，如果你用系统自带的 bash，需要改为 `.bash_profile`。

在 mac os 中，所有 JDK 安装在 `/Library/Java/JavaVirtualMachines/` 目录中：

```
blog (hexo) ✗ ll /Library/Java/JavaVirtualMachines/
total 0
drwxr-xr-x  3 root     wheel    96B  9 10 22:32 jdk-10.0.2.JDK
drwxr-xr-x  3 root     wheel    96B  9 10 22:20 jdk-9.0.4.JDK
drwxr-xr-x  3 root     wheel    96B  1 15  2016 jdk1.8.0_66.JDK
drwxr-xr-x@ 3 satansk  staff    96B 10  6 20:25 openjdk-11.0.1.JDK
```

可以看到我的系统中有 4 个版本的 JDK，除 Java 8/9/10 外，还有 mac os 自带的 openjdk。

因为 jenv 无法自动查找 JDK，所以需手动添加（or 删除）：

```
jenv add /Library/Java/JavaVirtualMachines/JDK1.8.0_66.JDK/Contents/Home
```

依次添加完成后，执行 `jenv rehash` 进行 jenv 相关的初始化工作。

最后可以查看 jenv 管理的所有版本：

```
blog (hexo) ✗ jenv versions
  system
  1.8
  1.8.0.66
  10.0
  10.0.2
  11.0
  11.0.1
  9.0
  9.0.4
  openJDK64-11.0.1
* oracle64-1.8.0.66 (set by /Users/satansk/.jenv/version)
  oracle64-10.0.2
  oracle64-9.0.4
```

`*` 表示当前的全局默认 JDK 版本，可通过 `jenv global` 修改：

```
jenv global oracle64-9.0.4
```

不同项目使用的默认 JDK 版本可能不同，可通过 `jenv local` 为每个项目设置自己的 JDK 版本：

```
jenv local oracle64-10.0.2
```

`jenv local` 在当前目录下创建 `.java-version` 文件，通过该文件为项目设置专门的 JDK 版本。

jenv 还可以为不同构建工具（如 sbt、maven、gradle）设置 JDK 版本，更多细节，请参考 [官方文档]()。

---

参考：

* [Multiple JDK versions on Mac OS X with jEnv](https://medium.com/@danielnenkov/multiple-jdk-versions-on-mac-os-x-with-jenv-5ea5522ddc9b)

