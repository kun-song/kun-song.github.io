---
title: emacs | 三个常用快捷键
date: 2018-12-05 22:58:01
tags:
  - emacs
categories: 编程工具
---

作为“神的编辑器”，emacs 以快捷键繁多著称，但即使资深用户也很难记清所有快捷键，不过好在 emacs 是 self-documenting 的，即 emacs 本身提供所有使用所需文档，当然也包括快捷键文档了。

重点在于三个快捷键：

* `C-h k`
* `C-h f`
* `C-h v`

<!-- more -->

举一个简单例子，如果我们想查看快捷键 `C-x C-f` 的文档，输入 `C-h k` 后 emacs 提示：

```
Describe the following key, mouse clike, or menu item:
```

即可以输入快捷键组合、鼠标事件、菜单选项等，emacs 将查询其文档，我们这里输入 `C-x C-f`，得到：

```
C-x C-f runs the command find-file (found in global-map), which is an
interactive compiled Lisp function in ‘files.el’.

It is bound to <open>, C-x C-f, <menu-bar> <file> <new-file>.

(find-file FILENAME &optional WILDCARDS)

...
```

原来 `C-x C-f` 绑定到 `find-file` 函数啊，那敲快捷键与直接输入函数效果相同：

```
M-x find-file
```
n
问题又来了，`find-file` 函数作用是什么呢？

敲 `C-h f`，emacs 提示：

```
Describe function (default find-file):
```

然后输入 `find-file`，得到 `find-file` 函数的文档：

```
find-file is an interactive compiled Lisp function in ‘files.el’.

It is bound to <open>, C-x C-f, <menu-bar> <file> <new-file>.

(find-file FILENAME &optional WILDCARDS)
```

最后一个 `C-h v` 用于查找变量定义，比如查找 `find-file-hook` 结果如下：

```
find-file-hook is a variable defined in ‘files.el’.
Its value is (ensime-run-find-file-hooks t)
Original value was nil
Local in buffer build.sbt; global value is 
(gdb-find-file-hook find-file-hook--open-junk-file url-handlers-set-buffer-mode global-eldoc-mode-check-buffers global-font-lock-mode-check-buffers epa-file-find-file-hook vc-refresh-state)

...
```

总结一下：

* `C-h k` 查找快捷键
* `C-h f` 查找函数
* `C-h v` 查找变量

熟练使用这三个快捷键对掌握 emacs 帮助很大！
