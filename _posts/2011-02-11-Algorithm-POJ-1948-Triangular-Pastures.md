---
layout: post
title: "POJ 1948 Triangular Pastures[二维01背包]"
keywords: ["algorithm", "POJ"]
description: "背包问题--POJ 1948 Triangular Pastures 解题报告"
category: "algorithm"
tags: ["ACM","dp"]
comments: true
---


[POJ 1948 Triangular Pastures](http://poj.org/problem?id=1948)

# 题目描述：
>给最多40根木棍，每根长度不超过40，
>要用完所有的木棍构成面积最大的三角形，
>求出最大的面积。

# 算法核心

二维01背包
 
***使用到海伦公式***：


  已知三角形的三边长度a,b,c,求面积
  S=√[p(p-a)(p-b)(p-c)] 
  而公式里的p为半周长： 
  p=(a+b+c)/2

# 分析：
用dp[i][j][k]表示到第i根木棒能否摆出边长分别为j,k的三角形
     易得 
     
``` 
dp[i][j][k] = dp[i-1][j-x[i]][k]|dp[i-1][j][k-x[i]]|dp[i-1][j][k];
```

  简单的
  空间压缩，化为二维dp，注意：这里每根木棒只能使用一次，
  是01背包，要倒着扫（刚开始错在这里了。。。）
  
# Answer

```
#include<stdio.h>
#include<string.h>
#include<math.h>
constint N =801;
bool dp[N][N];//dp[i][j]表示取两边分别为i,j可达
int x[N];
int main()
{
 int n;
 while(scanf("%d",&n)!=EOF)
 {
  int i,j,k;
  int sum =0;
  for(i=0;i<n;i++)
  {
   scanf("%d",&x[i]);
   sum+=x[i];
  }

  memset(dp,false,sizeof(dp));
  dp[0][0]=true;
  int half = sum>>1;
  for(i=0;i<n;i++)
   for(j=half;j>=0;j--)
    for(k=j;k>=0;k--)
    {
     if(x[i]<=j)
     dp[j][k]|=dp[j-x[i]][k];
     if(k>=x[i])
      dp[j][k]|=dp[j][k-x[i]];
    }
       double ha = sum/2.0;
    double ans =-1;
    for(i=0;i<half;i++)
     for(j=0;j<=i;j++)
     {
      if(dp[i][j])
      {
       k=sum-i-j;
       if(i+j>k&&i+k>j&&k+j>i)
       {
        double temp = ha*(ha-i)*(ha-j)*(ha-k);
        if(temp>ans)ans=temp;
       }
      }
     }

     intout;
     if(ans<0)out=-1;
        else
     out= (int)(sqrt(ans)*100);
 
   printf("%d\n",out);
 }
 return0;
}

```
