---
title: 模型和代价函数
date: 2018-1-13 15:34:04
categories:
- ai
tags:
- ai
- machine-learning
- 读书笔记
---

# 模型和代价函数

## 模型表示方法

约定x(i)表示输入变量(input variable)（即房地产例子中的房屋尺寸），y(i)表示输出或者目标变量(output/target variable)，即我们试图预测的（房价）。

一对(x(i), y(i))成为一个训练例。一系列我们用来机器学习的训练例的数据集，我们称之为训练集。

上标（我不会打上标，o(╥﹏╥)o）i表示训练集中的索引，与幂运算无关。

我们来更正式地描述监督式学习：我们的目标是，给定一个训练集，来学习得一个函数h: X -> Y，以便h(x)能够很好地预测出对应的y值。由于历史原因，这个函数被称为假设(hypothesis)。

![supervised-learning-process](/dev-log/images/ai/supervised-learning-process.png)

当我们试图预测的目标变量是连续的，就像我们的房价预测问题，我们将这个学习问题成为回归问题。

当y只能取少数几个离散值，我们称之为分类问题。

## 代价函数

我们可以通过代价函数(cost function)来衡量假设函数(hypothesis function)的精确性。
