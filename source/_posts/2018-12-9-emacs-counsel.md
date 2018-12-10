---
title: emacs | counsel 插件
date: 2018-12-09 23:16:11
tags:
  - emacs
  - counsel
  - swiper
  - Ivy
categories: 编程工具
---

虽然题目只提到 counsel，但实际是 Ivy、Counsel、Swiper 三个插件的组合，它们在 [同一个 github 仓库](https://github.com/abo-abo/swiper/tree/f87962631101d3db12c86de3f71f0b21c85896c0) 中，各自角色如下：

* Ivy, a generic completion mechanism for Emacs.
* Counsel, a collection of Ivy-enhanced versions of common Emacs commands.
* Swiper, an Ivy-enhanced alternative to isearch.

简单说 Ivy 是 emacs 的一种通用补全机制，Counsel 和 Swiper 建立在 Ivy 之上，提供 emacs 默认命令的增强版本。

<!-- more -->

emacs 有很多内置功能，比如 `M-x` 执行命令、`C-s` 搜索等，这些功能很常用，但它们都不够好用，起码自动补全做的不好，counsel 提供的功能比内置版强大、好用，非常推荐。

通过 `M-x package-install` 从 MELPA 安装，安装后简单配置如下：

```
(ivy-mode 1)
(setq ivy-use-virtual-buffers t)
(setq enable-recursive-minibuffers t)
(global-set-key "\C-s" 'swiper)
(global-set-key (kbd "C-c C-r") 'ivy-resume)
(global-set-key (kbd "<f6>") 'ivy-resume)
(global-set-key (kbd "M-x") 'counsel-M-x)
(global-set-key (kbd "C-x C-f") 'counsel-find-file)
(global-set-key (kbd "C-h f") 'counsel-describe-function)
(global-set-key (kbd "C-h v") 'counsel-describe-variable)
(global-set-key (kbd "C-h l") 'counsel-find-library)
(global-set-key (kbd "<f2> i") 'counsel-info-lookup-symbol)
(global-set-key (kbd "<f2> u") 'counsel-unicode-char)
(global-set-key (kbd "C-c g") 'counsel-git)
(global-set-key (kbd "C-c j") 'counsel-git-grep)
(global-set-key (kbd "C-c k") 'counsel-ag)
(global-set-key (kbd "C-x l") 'counsel-locate)
(global-set-key (kbd "C-S-o") 'counsel-rhythmbox)
(define-key minibuffer-local-map (kbd "C-r") 'counsel-minibuffer-history)
```

现在 `C-s` 搜索绑定到 `'swiper` 命令，搜索时它会提供一个小小的预览窗口，并可通过 `C-n/p` 上下滑动选择：

<img src="/images/emacs/counsel/swiper-1.png" alt="swiper 搜索界面" />

而且 `M-x` 现在也有了自动补全，比如配置插件时：

<img src="/images/emacs/counsel/m-x-1.png" alt="M-x 自动提示" />

我们还绑定了很多其他 counsel 函数的快捷键，比如 `counsel-find-file`、`counsel-describe-function`、`counsel-ag` 等等，它们真的比 emacs 内置版好用太多，自动补全非常快捷，细节设计周到。

counsel 可以大大改善 emacs 使用体验，十分推荐！

