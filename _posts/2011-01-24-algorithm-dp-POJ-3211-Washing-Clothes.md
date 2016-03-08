---
layout: post
title: "POJ 3211 Washing Clothes [背包]"
keywords: ["algorithm", "背包","POJ"]
description: "POJ 3211 Washing Clothes [背包]解题报告"
category: "algorithm"
tags: ["ACM","背包","DP"]
comments: true
---

[POJ 3211 Washing Clothes](http://poj.org/problem?id=3211)

## 核心算法：
01背包

## 解题思路： 
首先按颜色对衣服进行归类，即将相同颜色的衣服放在同一类中
对于某一种颜色的所有衣服所需要的最少时间，相当于将这堆衣服按时间分为两推，使得这两堆衣服所需要的时间尽可能的接近。

### 对于每堆衣服建模：
   假设当前这堆衣服一个人洗的时间为sum, 令mid = sum/2;
   
**问题转化为**

（1）有背包容量为mid,现在要从这堆衣服中选取衣服，使得总容量尽可能接近于mid

**继续转化..**

（2）背包容量为mid,某件衣服的重量为wi,价值也为wi,计算所能达到的最大价值 dp[mid].

  那么问题（2）中的dp[mid]相当于问题（1）中最接近于mid的那个容量，故原问题中这堆衣服所需要的实际时间为***sum - dp[mid]***;
  
## Answer

```c++
#include<stdio.h>
#include<string.h>
#include<vector>
usingnamespace std;
constint M =10;
constint N =100;
char color[M][N];//存储颜色,及其对应的id
vector<int>cost[M];//存储每种颜色对应的衣物所需时间
int dp[10000+2];

inline int max(int a,int b)
{
    return a>b?a:b;
}

int main()
{
    int n,m;
    while(scanf("%d%d",&m,&n)!=EOF)
    {
        if(m==0&&n==0)break;
        int i,j;
        for(i=0;i<m;i++)
        {
            scanf("%s",color[i]);//将每一种颜色输入其中
            cost[i].clear();
        }

        while(n--) //输入每一件衣服的基本信息
        {
            char col[N];
            int time;
            scanf("%d%s",&time,col);
            for(i=0;i<m;i++)
                if(strcmp(col,color[i])==0)break;
        
           if(i<m)  //将当前衣服信息放入对应的颜色内
           {
              cost[i].push_back(time);
           }
        }

        int ans =0;
        for(i=0;i<m;i++)   //对每一种颜色的衣服进行01背包建模
        {
           memset(dp,0,sizeof(dp));
           int sz = cost[i].size();
           int k,mid ,sum =0;
           for(j=0;j<sz;j++)sum += cost[i][j];
           mid = sum/2;
           for(j=0;j<sz;j++)
           {
               for(k = mid;k>=cost[i][j];k--)
               {
                   dp[k] = max(dp[k],dp[k-cost[i][j]]+cost[i][j]);
               }    
           }
           ans = ans + sum - dp[mid];
        }
        printf("%d\n",ans);
    }
    return0;
}
```
