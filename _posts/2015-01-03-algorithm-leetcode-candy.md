---
layout: post
title: "leetcode-gas-station"
keywords: ["greedy", "leetcode"]
description: "leetcode-candy"
category: "algorithm"
tags: ["algorithm","gas-station"]

---
#[Gas Station](https://oj.leetcode.com/problems/gas-station/)
##大意
There are N gas stations along a circular route, where the amount of gas at station i is gas[i].

You have a car with an unlimited gas tank and it costs cost[i] of gas to travel from station i to its next station (i+1). You begin the journey with an empty tank at one of the gas stations.

Return the starting gas station's index if you can travel around the circuit once, otherwise return -1.

Note:
The solution is guaranteed to be unique.

##简要思路
从起点开始挨个枚举各个节点作为起点，如果遇到油量不足，下一次枚举点从该点的下一节点开始。

##证明
设以下变量

* left[i]为从i点出发到下一节点、且未加终点油前的剩余油量。
* sum[i][j] 为从点i出发到j点，且未加j点油之前的剩余油量。

那么问题演化为:

```
求最小的i值，使得对于i+1...n..0..i-1
	所有的sum[i][k] >= 0
```
	

基于以上变量有以下等式成立：

1. left[i]  = gas[i]-cost[i]
2. sum[i][j] = sum[i][j-1]+left[j]
3. sum[i][j] = sum[i][k] + sum[k][j];




从0开始枚举各起点
对于某点i作为起始点，假设到第j点以前的剩余油量均大于等于0，即：

* sum[i][k]>=0 {i<=k<j}

而到第j点时sum[i][j] < 0, 即：

* sum[i][j] = sum[i][j-1] + left[j] < 0

那么对于k>=i&&k<j：

* sum[k][j] = sum[i][j] - sum[i][k] <= sum[i][j] < 0 

即对于所有i到j之间的点k,sum[k][j] < 0,故下一满足条件的起点至少为j+1. 

## Code
[gas_station.cpp](https://github.com/AndreMouche/algorithms_study/blob/master/leetcode/gas_station.cpp)