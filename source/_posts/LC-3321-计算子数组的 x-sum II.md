---
title: 3321.计算子数组的 x-sum II
date: 2025-11-04 18:16:13
tags:
- 对顶堆
---

## 题目描述
[leetcode Hard](https://leetcode.cn/problems/find-x-sum-of-all-k-long-subarrays-ii/)

给你一个由 n 个整数组成的数组 nums，以及两个整数 k 和 x。

数组的 x-sum 计算按照以下步骤进行：

- 统计数组中所有元素的出现次数。
- 仅保留出现次数最多的前 x 个元素的每次出现。如果两个元素的出现次数相同，则数值 较大 的元素被认为出现次数更多。
- 计算结果数组的和。

注意，如果数组中的不同元素少于 x 个，则其 x-sum 是数组的元素总和。


返回一个长度为 n - k + 1 的整数数组 answer，其中 answer[i] 是 子数组 nums[i..i + k - 1] 的 x-sum。

子数组 是数组内的一个连续 非空 的元素序列。

示例1：
```
输入：nums = [1,1,2,2,3,4,2,3], k = 6, x = 2

输出：[6,10,12]

解释：

对于子数组 [1, 1, 2, 2, 3, 4]，只保留元素 1 和 2。因此，answer[0] = 1 + 1 + 2 + 2。
对于子数组 [1, 2, 2, 3, 4, 2]，只保留元素 2 和 4。因此，answer[1] = 2 + 2 + 2 + 4。注意 4 被保留是因为其数值大于出现其他出现次数相同的元素（3 和 1）。
对于子数组 [2, 2, 3, 4, 2, 3]，只保留元素 2 和 3。因此，answer[2] = 2 + 2 + 2 + 3 + 3。
```

提示1：
```
nums.length == n
1 <= n <= 10^5
1 <= nums[i] <= 10^9
1 <= x <= k <= nums.length
```

## 对顶堆
实际上是求滑动窗口的前 $x$ 大元素的和（这里的比较条件是出现次数）。

类似 [对顶堆](https://oi-wiki.org/ds/binary-heap/#%E5%AF%B9%E9%A1%B6%E5%A0%86) 的思路，只需要将比较条件改成计数以及额外再维护一个 $sum$ ，但注意到 Java 中堆删除一个指定元素是慢的，所以使用TreeMap来实现堆的功能。


时间复杂度 $O(NlogK)$ 。
```Java
class Solution {
    private final Map<Integer, Integer> cnt = new HashMap<>();
    private final TreeSet<Integer> MIN = new TreeSet<>((a, b) -> {
        int cntSub = cnt.get(a) - cnt.get(b);
        if (cntSub != 0) {
            return cntSub;
        }
        return a - b;
    });
    private final TreeSet<Integer> MAX = new TreeSet<>(Objects.requireNonNull(MIN.comparator()).reversed());
    private long sum = 0;

    public long[] findXSum(int[] nums, int k, int x) {
        long[] ans = new long[nums.length - k + 1];
        for (int r = 0; r < nums.length; r++) {
            // 刷新nums[r]
            int in = nums[r];
            del(in);
            cnt.merge(in, 1, Integer::sum);
            add(in);

            int l = r + 1 - k;
            if (l < 0) {
                continue;
            }

            // 维护堆大小
            while (!MAX.isEmpty() && MIN.size() < x) {
                r2l();
            }
            while (MIN.size() > x) {
                l2r();
            }
            ans[l] = sum;

            // 刷新nums[l]
            int out = nums[l];
            del(out);
            cnt.merge(out, -1, Integer::sum);
            add(out);
        }
        return ans;
    }

    private void add(int val) {
        int c = cnt.get(val);
        if (c == 0) {
            return;
        }
        if (!MIN.isEmpty() &&
                Objects.requireNonNull(MIN.comparator()).compare(val, MIN.first()) > 0) {
            sum += (long) val * c;
            MIN.add(val);
        } else {
            MAX.add(val);
        }
    }

    private void del(int val) {
        int c = cnt.getOrDefault(val, 0);
        if (c == 0) {
            return;
        }
        if (MIN.contains(val)) {
            sum -= (long) val * c;
            MIN.remove(val);
        } else {
            MAX.remove(val);
        }
    }

    private void l2r() {
        int p = Objects.requireNonNull(MIN.pollFirst());
        sum -= (long) p * cnt.get(p);
        MAX.add(p);
    }

    private void r2l() {
        int p = Objects.requireNonNull(MAX.pollFirst());
        sum += (long) p * cnt.get(p);
        MIN.add(p);
    }
}
```