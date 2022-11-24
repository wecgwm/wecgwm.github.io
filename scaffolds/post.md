---
title: {{ title }}
date: {{ date }}
tags:
---

## 题目描述
[leetcode 中等题]()



示例1：
```
```

提示1：
```
```

## 动态规划

$$
\begin{align}
&dp(i, j) = \frac{1}{4} \times (dp(i - 4, j) + dp(i - 3, j - 1) + dp(i - 2, j - 2) + dp(i - 1, j - 3))\\\\
\end{align}
$$
