---
title: LC-2573.找出对应LCP矩阵的字符串
date: 2023-02-21 06:39:49
tags:
- 并查集
---

## 题目描述
[leetcode 困难题](https://leetcode.cn/problems/find-the-string-with-lcp/submissions/)

对任一由 n 个小写英文字母组成的字符串 word ，我们可以定义一个 n x n 的矩阵，并满足：

lcp[i][j] 等于子字符串 word[i,...,n-1] 和 word[j,...,n-1] 之间的最长公共前缀的长度。
给你一个 n x n 的矩阵 lcp 。返回与 lcp 对应的、按字典序最小的字符串 word 。如果不存在这样的字符串，则返回空字符串。

示例1：
```
输入：lcp = [[4,0,2,0],[0,3,0,1],[2,0,2,0],[0,1,0,1]]
输出："abab"
解释：lcp 对应由两个交替字母组成的任意 4 字母字符串，字典序最小的是 "abab" 。
```

提示1：
```
1 <= n == lcp.length == lcp[i].length <= 1000
0 <= lcp[i][j] <= n
```

## 并查集
该题的关键是注意到 $lcp$ 实际上给出了字符串任意两个下标 $i、j$ 是否相同的信息。也就是说在构造的过程中，我们可以暂时只关注 $lcp[i][j]$ 是否大于 $0$ ，忽略究竟有多长，而因为 $i、j$ 涵盖了所有下标，导致我们可以构造出一个唯一的目标字符串。然后再通过该字符串推出一个新的 $lcp$ 矩阵，最后对比两个矩阵，如果相同则证明原矩阵给出的 $lcp[i][j]$ 的长度值是正确的。

对于构造目标字符串我们可以用并查集来将所有 $lcp[i][j] > 0$ 的点连通，最后再按照字典序将每个连通块填上相同的最小字母即可。

而对于求某个字符串的 $lcp$ ，我们可以通过动态规划实现，具体如下：
$$
\begin{align}
&dp(i, j) = dp[i + 1][j + 1] + 1　　　　　　　　　　　　　　　　　　　 if　s[i] == s[j]\\\\
&dp(i, j) = 0　　　　 　　　　　　　　　　　　　　　　　　　　　　　     else \\\\
\end{align}
$$

```Java
class Solution {
    public String findTheString(int[][] lcp) {
        int n = lcp.length;
        UnionFind unionFind = new UnionFind(n);
        for(int i = 0; i < n; i++){
            for(int j = 0; j < n; j++){
                if(lcp[i][j] > 0){
                    unionFind.union(i, j);
                }
            }
        }
        Map<Integer, List<Integer>> map = unionFind.group();
        char minChar = 'a';
        StringBuilder ans = new StringBuilder("*".repeat(n));
        for (int i = 0; i < n; i++) {
            if (ans.charAt(i) != '*') {
                continue;
            }
            if (minChar == ('z' + 1)) {
                return "";
            }
            for (Integer index : map.get(unionFind.find(i))) {
                ans.setCharAt(index, minChar);
            }
            minChar = (char) (minChar + 1);
        }
        int[][] check = new int[n][n];
        for (int i = n - 1; i >= 0; i--) {
            for (int j = n - 1; j >= 0; j--) {
                if (i + 1 > n - 1 || j + 1 > n - 1) {
                    check[i][j] = ans.charAt(i) == ans.charAt(j) ? 1 : 0;
                }else{
                    check[i][j] = ans.charAt(i) == ans.charAt(j) ? check[i + 1][j + 1] + 1 : 0;
                }
            }
        }
        return Arrays.deepEquals(check, lcp) ? ans.toString() : "";
    }
}

class UnionFind{
    int[] fa;
    int[] size;

    public UnionFind(int n){
        fa = IntStream.range(0, n).toArray();
        size = IntStream.generate(() -> 1).limit(n).toArray();
    }

    public void union(int a, int b){
        a = find(a);
        b = find(b);
        if(a == b){
            return;
        }
        if(size[a] > size[b]){
            a = a ^ b;
            b = a ^ b;
            a = a ^ b;
        }
        fa[b] = a;
        size[a] += size[b];
    }

    public int find(int a){
        if(a != fa[a]){
            fa[a] = find(fa[a]);
        }
        return fa[a];
    }

    public Map<Integer, List<Integer>> group() {
        Map<Integer, List<Integer>> ret = new HashMap<>();
        for (int i = 0; i < size.length; i++) {
            ret.computeIfAbsent(fa[i], k -> new ArrayList<>())
                    .add(i);
        }
        return ret;
    }
}
```
类似的思路但没有用到并查集
```Java
class Solution {
    public String findTheString(int[][] lcp) {
        int i = 0, n = lcp.length;
        var s = new char[n];
        for (char c = 'a'; c <= 'z'; ++c) {
            while (i < n && s[i] > 0) ++i;
            if (i == n) break; // 构造完毕
            for (int j = i; j < n; ++j)
                if (lcp[i][j] > 0) s[j] = c;
        }
        while (i < n) if (s[i++] == 0) return ""; // 没有构造完

        // 直接在原数组上验证
        for (i = n - 1; i >= 0; --i)
            for (int j = n - 1; j >= 0; --j) {
                int actualLCP = s[i] != s[j] ? 0 : i == n - 1 || j == n - 1 ? 1 : lcp[i + 1][j + 1] + 1;
                if (lcp[i][j] != actualLCP) return "";
            }
        return new String(s);
    }
}
```