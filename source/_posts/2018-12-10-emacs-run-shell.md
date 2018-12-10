---
title: emacs | 在 emacs 中运行 shell
date: 2018-12-10 23:38:41
tags:
  - emacs
  - shell
categories: 编程工具
---

我自己使用 emacs 时常常需要运行 shell 脚本，这时可以切换到 terminal 窗口，操作完成后再切换回 emacs，这样当然可以，但更好的方式是在 emacs 中直接运行 shell。

emacs 中有 3 种运行 shell 的方式，下面一一介绍。

<!-- more -->

## shell mode

运行 `M-x shell` 指令将得到一个系统默认 shell 的 wrapper，该 wrapper 只是“傻瓜地”在默认 shell 和 emacs 之间传递输入输出内容，因此交互类的工具（`ssh`、`top` 等）将无法工作。

<img src="/images/emacs/shell/shell-mode.png" alt="shell mode" />

在 macbook pro 上 `clear` 指令无法清除该 shell 屏幕，而且 `C-n/p` 也无法上下切换，用户体验很不好。

## terminal emulator

我们平时用的 xterm 等都是 terminal 模拟器，emacs 也提供了一个，通过 `M-x term/acsi-term` 调用：

<img src="/images/emacs/shell/terminal-emulator.png" alt="termimal" />

emacs terminal 模拟器比 shell 模式强大很多，像 `top/clear` 等工具（甚至 `vim`）都可以正常工作，但它并不完美，部分需要交互的指令可能无法正常工作。

注意 emacs terminal emulator 快捷键与 iterm 等系统自带的 emulator 相同，因此会与 emacs 快捷键冲突，emacs 通过两种模式区分两者：

* line-mode：普通 emacs buffer；
* char-mode：用户输入直接传递给 terminal emulator，不经过 emacs 处理；

>`C-c C-j` 切换到 line-mode，`C-c C-k` 切换到 char-mode。

## eshell

世界上有很多 shell，比如 Linux 常用的 bash，以及大家都很喜欢的 zsh 等等，emacs 也提供了名为 eshell 的实现，它是用 Emacs-Lisp 编写的，通过 `M-x eshell` 调用：

<img src="/images/emacs/shell/eshell.png" alt="eshell" />

上图是在 eshell 中打开 vim 的界面，可以看到已经乱码 :(

eshell 使用并不广泛，不过在 windows 中可以尝试下，毕竟 windows 没有 termimal。

## 总结

上面三种方式可以从 buffer 名字区分：

* shell wrapper buffer 名字为 shell
* termimal emulator buffer 名字为 termimal
* eshell buffer 名字为 eshell

其中最实用的是 termimal emulator。

---

参考：

* [Running Shells in emacs: An Overview](https://www.masteringemacs.org/article/running-shells-in-emacs-overview)

