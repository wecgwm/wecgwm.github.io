---
title: LC-375.猜数字大小II
date: 2022-11-05 10:47:17
tags:
- 记忆化搜索
- 区间DP
---

## 题目描述
[leetcode 中等题](https://leetcode.cn/problems/guess-number-higher-or-lower-ii/)

我们正在玩一个猜数游戏，游戏规则如下：

我从 1 到 n 之间选择一个数字。
1. 你来猜我选了哪个数字。
2. 如果你猜到正确的数字，就会赢得游戏 。
3. 如果你猜错了，那么我会告诉你，我选的数字比你的 更大或者更小 ，并且你需要继续猜数。
4. 每当你猜了数字 x 并且猜错了的时候，你需要支付金额为 x 的现金。如果你花光了钱，就会 输掉游戏 。

给你一个特定的数字 n ，返回能够 确保你获胜 的最小现金数，不管我选择那个数字 。

示例1：
```
输入：n = 2
输出：1
解释：有两个可能的数字 1 和 2 。
- 你可以先猜 1 。
    - 如果这是我选中的数字，你的总费用为 $0 。否则，你需要支付 $1 。
    - 如果我的数字更大，那么这个数字一定是 2 。你猜测数字为 2 并赢得游戏，总费用为 $1 。
最糟糕的情况下，你需要支付 $1 。
```

示例2：
```
输入：n = 1
输出：0
解释：只有一个可能的数字，所以你可以直接猜 1 并赢得游戏，无需支付任何费用。
```

提示:
```
1 <= n <= 200
```

## 递归(TLE)
枚举所有选择，以及对应答案的所有可能，稳 TLE，需要优化。
```Java
class Solution {
    public int getMoneyAmount(int n) {
        return dfs(1, n);
    }

    private int dfs(int start, int end){
        if(start >= end){
            return 0;
        }
        int ans = Integer.MAX_VALUE;
        for(int i = start; i <= end; i++){
            // 无法决策答案是哪个数，只能取最差情况，因为要确保获胜
            int cur = Math.max(dfs(start, i - 1), dfs(i + 1, end)) + i;
            // 可以决策要猜哪个数，取最好的情况
            ans = Math.min(ans, cur);
        }
        return ans;
    }
}
```

## 记忆化搜索
容易发现计算的结果其实只跟区间的开始以及结束有关（即 dfs 的入参），同时又因为数据范围只有 1-200 ，所以可以通过一个二维数组 `cache` 来保存计算过的结果来避免重复计算，`cache[i][j]` 表示 i 到 j 范围的数确保获胜的最小现金数， `cache[1][n]` 为题目所求。
```Java
class Solution {
    static int[][] cache;

    public int getMoneyAmount(int n) {
        cache = new int[n + 1][n + 1];
        return dfs(1, n);
    }

    private int dfs(int start, int end){
        if(start >= end){
            return 0;
        }
        if(cache[start][end] != 0){
            return cache[start][end];
        }
        int ans = Integer.MAX_VALUE;
        for(int i = start; i <= end; i++){
            // 无法决策答案是哪个数，只能取最差情况，因为要确保获胜
            int cur = Math.max(dfs(start, i - 1), dfs(i + 1, end)) + i;
            // 可以决策要猜哪个数，取最好的情况
            ans = Math.min(ans, cur);
        }
        cache[start][end] = ans;
        return ans;
    }
}
```

##　区间 DP

我们发现在求解 `[start, end]` 区间时，假设当前选择的数是 `i`，那么只会依赖区间 `[start, i - 1]` 和 `[ i + 1, end]`，同时还具有以下几点性质：
1. 每次在求解某个区间的结果时，只会依赖更小的区间
2. `f(start, end)` 下某个 `i` 最小成本 = `max(f(start, i - 1), f(i + 1, end)) + i`
3. 如果 `start == end`，那么最小成本为 0，如果 `start + 1 == end`, 那么最小成本为 `start`

由第 1 点可知在求解区间需要逆推，从 `[n - 2, n]` 开始扩散区间直到求出 `[1, n]`，整个过程如下：

`[n - 2, n - 2 + 2] -> [n - 3, n - 1] -> [n - 3, n] -> ... -> [1, n]` 

而由第 3 点我们可以先得到所有 `start + 1 <= end` 的区间结果，那么在求解其他所有区间的过程中就可以通过第 2 点的式子以及第 3 点的结果逐步得出所有区间的结果。
```Java
class Solution {

    public int getMoneyAmount(int n) {
        int[][] dp = new int[n + 2][n + 2];
        for(int i = 1; i < n; i++){
             dp[i][i + 1] = i; // 初始化所有 start + 1 <= end 区间的结果
        }
        for(int i = n - 2; i >= 1; i--){ // 从小区间开始逐渐扩散
            for(int j = i + 2; j <= n; j++){ 
                int cur = Integer.MAX_VALUE;
                for(int x = i; x <= j; x++){ // 枚举猜的数
                    int t = Math.max(dp[i][x - 1], dp[x + 1][j]) + x; // 无法决策答案是哪个数，只能取最差情况，因为要确保获胜
                    cur = Math.min(cur, t); // 取枚举出来的猜某个数的最好结果
                }
                dp[i][j] = cur;
            }
        }
        return dp[1][n];
    }

}
```