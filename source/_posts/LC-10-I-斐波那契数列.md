---
title: LC-10-I.斐波那契数列
date: 2022-11-04 23:08:04
tags:
- 矩阵快速幂
---

## 题目描述
[leetcode 简单题](https://leetcode.cn/problems/fei-bo-na-qi-shu-lie-lcof/)

写一个函数，输入 n ，求斐波那契（Fibonacci）数列的第 n 项（即 F(N)）。斐波那契数列的定义如下：

```
F(0) = 0,   F(1) = 1

F(N) = F(N - 1) + F(N - 2), 其中 N > 1.
```

斐波那契数列由 0 和 1 开始，之后的斐波那契数就是由之前的两数相加而得出。

答案需要取模 1e9+7（1000000007）

## 朴素解法
```Java
class Solution {
    static int MOD = (int)(1E9 + 7);

    public int fib(int n) {
        if(n <= 1){
            return n;
        }
        int a = 0, b = 1;
        for(int i = 2; i <= n; i++){
            int temp = ((a % MOD) + (b % MOD)) % MOD; // 同余
            a = b;
            b = temp;
        }
        return b;
    }

}
```

## 矩阵快速幂
对于数列递推问题，可以使用矩阵快速幂进行加速，矩阵快速幂的时间复杂度能够突破线性达到 `O(logN)`。

[OI-WIKI-快速幂](https://oi-wiki.org/math/binary-exponentiation/)　　[OI-WIKI-矩阵乘法](https://oi-wiki.org/math/linear-algebra/matrix/#%E7%9F%A9%E9%98%B5%E4%B9%98%E6%B3%95)　　[宫水三叶-矩阵快速幂](https://mp.weixin.qq.com/s?__biz=MzU4NDE3MTEyMA==&mid=2247488198&idx=1&sn=8272ca6b0ef6530413da4a270abb68bc&chksm=fd9cb9d9caeb30cf6c2defab0f5204adc158969d64418916e306f6bf50ae0c38518d4e4ba146&token=1067450240&lang=zh_CN#rd)

```Java
class Solution {
    static int MOD = (int)(1E9 + 7);

    public int fib(int n) {
        if (n <= 1) {
            return n;
        }
        long[][] matrix = {
                {1, 1},
                {1, 0}
        };
        long[][] ans = {
            {1, 0},
            {0, 1}
        }; // 矩阵中的1，对角线为1
        int k = n - 1;
        while (k != 0) {
            if ((k & 1) != 0) {
                ans = mul(matrix, ans); // 快速幂，将对应二进制位为 1 时的整系数幂乘起来
            }
            matrix = mul(matrix, matrix);
            k >>= 1;
        }
        return (int)(ans[0][0] % MOD); // 实际上为 ans[0][0] * f(1) + ans[0][1] * f(0)
    }

    private long[][] mul(long[][] matrix1, long[][] matrix2) {
        int n = matrix1.length;
        long[][] ret = new long[n][n];
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                ret[i][j] = (((matrix1[i][0] * matrix2[0][j]) % MOD) + ((matrix1[i][1] * matrix2[1][j]) % MOD)) % MOD;
            }
        }
        return ret;
    }
}
```