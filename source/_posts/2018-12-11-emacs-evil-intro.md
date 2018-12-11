---
title: emacs | Evil 插件
date: 2018-12-11 23:08:54
tags:
  - emacs
  - Evil
categories: 编程工具
---

很多 emacs 用户其实非常眼馋 vim 的强大编辑能力，包括我，好在有 Evil，它在 emacs 中几乎完全“模拟”了 vim 的编辑功能，在 emacs 中模拟 vim，的确非常“邪恶”。

通过 `M-x` 从 MELPA 安装 Evil，安装完成后通过 `C-h i` 可查看 Evil 的文档：

<img src="/images/emacs/evil/evil-doc-1.png" alt="evil doc" />

<!-- more -->

开启 Evil 模式：

```
(require 'evil)
(evil-mode 1)
```

另外，文档提到：

```
Evil requires 'undo-tree.el' to provide linear undo and undo branches.
It is available from EmacsWiki.(1)  (A copy of 'undo-tree.el' is also
included in the Git repository.)
```

即 Evil 部分功能需要 undo-tree 插件支持，同样通过 `M-x` 安装。

重启 emacs 后，Evil 即可生效：

<img src="/images/emacs/evil/evil-1.png" alt="evil doc" />

注意状态栏的 `<N>` 标志，它表示当前处于 Normal State，对应于 vim 中的 Normal Mode，因为 emacs 已经有 mode 的概念了，所以这里改称 state，实际是一回事。

除 Normail State 外，Evil 还有 Insert State、Visual State、Replace State 等，这些 state 与 vim 中的各种 mode 一一对应，注意除 Normal State 外，其他 Evil state 中是可以用 emacs 快捷键的。

最后还有 Emacs State，即普通 emacs 模式，可通过 `C-z` 在 Normal State 和 Emacs State 之间切换，非常方便。

不过还有一点瑕疵，即安装 Evil 后编辑时光标（cursor）变成方块了，而我的配置文件中专门将光标设置为了 bar：

```
(setq-default cursor-type 'bar)
```

该配置貌似失效了，Evil 文档中专门有一节介绍光标设置：

```
2.1 The cursor
==============

A state may change the cursor's appearance.  The cursor settings are
stored in the variables below, which may contain a cursor type as per
the 'cursor-type' variable, a color string as passed to the
'set-cursor-color' function, a zero-argument function for changing the
cursor, or a list of the above.  For example, the following changes the
cursor in Replace state to a red box:

     (setq evil-replace-state-cursor '("red" box))

If the state does not specify a cursor, 'evil-default-cursor' is used.

 -- Variable: evil-default-cursor
     The default cursor.

 -- Variable: evil-normal-state-cursor
     The cursor for Normal state.

 -- Variable: evil-insert-state-cursor
     The cursor for Insert state.

...
```

即 Evil 各 state 可以配置不同的光标形状、颜色，如果不配置，则默认用 `evil-default-cursor`，我把 Insert State 光标设为 bar，与 vim 保持一致：

```
(setq evil-insert-state-cursor 'bar)
```

到此为止，Evil 基本可以满足我的需求了：

<img src="/images/emacs/evil/final-evil.png" alt="evil final" />

最后，enjoy emacs and vim！

