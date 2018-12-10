---
title: emacs | 主题设置
date: 2018-12-11 00:50:20
tags:
  - emacs
  - theme
categories: 编程工具
---

N 年前我安装完 emacs，打开的默认界面是这样的：

<img src="/images/emacs/themes/emacs-raw.png" alt="shell mode" />

<!-- more -->

折腾了半天也没找到在哪里设置主题，当时我挺失望的：emacs 果然是个老古董！

当然，我当年太嫩了，还无法理解 emacs 的可玩性，直到今天我才真正开始理解和使用 emacs，主题设置这种事早已不再话下。

首先 `M-x` 安装需要的主题，然后两行代码就能完成基本设置：

```Lisp
(add-to-list 'my/packages 'spacemacs-theme)
(load-theme 'spacemacs-dark t)
```

更多配置可通过 `M-x customize-group  spacemacs-theme` 完成。

最后界面如下：

<img src="/images/emacs/themes/emacs-spacemacs.png" alt="shell mode" />

我还挺喜欢 spacemacs 主题的 :)
