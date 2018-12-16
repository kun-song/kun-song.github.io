---
title: 机器学习 | overfitting
date: 2018-12-15 21:13:43
tags:
  - 机器学习
  - underfitting
  - overfitting
categories: 机器学习
---

监督算法通过训练集训练算法，然后借助训练出的算法预测新数据，因此算法表现可以是：

* 训练集、新数据上都非常差
* 训练集、新数据都表现良好
* 训练集上表现优异，但预测新数据时效果非常差

理想的算法应该是第二种。

第一种算法被称为 underfitting/high bias，常见原因为：

* 过于简单的 hypothesis 函数
* hypothesis 函数使用的特征过少

<!-- more -->

第三种被称为 overfitting/high variance，即过拟合，原因为：

* 过于复杂的 hypothesis 函数

underfitting 一般不是问题，因为实际应用中一般不会使用太简单的 hypothesis 函数，大家都唯恐自己的函数不够复杂，因为需要解决的是 overfitting。

overfitting 有两类解决方式：

* 减少特征数量
  + 手动选择需要的特性
  + 使用 model selection 算法自动选择特征
* Regularization
  + 保持所有特征，但减少每个特征的权重（ `$\theta_j$` ）
  + 当有很多特征，但每个特征影响都不大的时候 regularization 表现良好

---

参考：

* [Andrew Ng 机器学习之 overfitting](https://www.coursera.org/learn/machine-learning/lecture/ACpTQ/the-problem-of-overfitting)

