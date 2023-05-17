---
title: {{ title }}
date: {{ date }}
tags:
---

## 题目描述
[leetcode hard]()



示例1：
```
```

提示1：
```
```

## 动态规划

$$
\begin{align}
&dp(i, j) = \frac{sum(nums[0] ... nums[i - 1])}{i}　　　　　　　　　　　　　　　　　　　 j = 1\\\\
&dp(i, j) = max\{dp(x, j - 1) + \frac{sum(nums[x + 1] ... nums[i])}{i - x}\}　　　　 1 <j ,　j - 1 <= x < i \\\\
\end{align}
$$
