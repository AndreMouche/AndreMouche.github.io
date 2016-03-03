---
layout: post
title: "leetcode-candy"
keywords: ["dp", "leetcode"]
description: "leetcode-candy"
category: "algorithm"
tags: ["algorithm","leetcode"]

---

[candy](https://oj.leetcode.com/problems/candy/)
##大意
有n个小伙伴排成一列，每个小伙伴有一个评分，先给其发蛋糕，要求满足以下条件：

1. 每人至少有一个蛋糕
2. 评分较高的人所拥有的糖果数比其友邻大。

问：最少需要多少糖果给这些人？

##思路
DP+Greedy
空间复杂度O(N)，时间复杂度O(N)

dp[i][0]表示第i左侧有多少数字连续递减的
```
if(ratings[i]>ratings[i-1]) {
  dp[i][0] = dp[i-1]+1
}else {
  dp[i][0] = 0
}
```
那么第i位置要满足左邻条件的糖果数至少为dp[i][0]+1

同样的，dp[i][1]表示i右侧有多少数字连续递减的，则

```
if(ratings[i]>ratings[i+1]) {
  dp[i][1] = dp[i+1][1]+1
} else {
  dp[i][1] = 0;
}
```
同样的，第i位置要满足友邻条件需要的糖果数至少为dp[i][1]+1
那么每个位置的最小蛋糕数为

```
ans[i] = max(dp[i][0],dp[i][1]) +1
```

证明：

假设r[i]<r[i+1],那么

1. dp[i+1][0] = dp[i][0]+1
2. dp[i][1] = 0,dp[i+1][1]>=0
得到：
1. ans[i] = dp[i][0] + 1
2. ans[i+1] = max(dp[i][0]+1,dp[i+1][1]) +1
无论如何，ans[i+1]>ans[i] ，满足条件2.

[详细代码]()