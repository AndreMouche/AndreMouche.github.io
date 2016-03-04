---
layout: post
title: "硬币找钱问题"
keywords: ["algorithm", "算法设计与分析"]
description: "硬币找钱问题解题报告"
category: "algorithm"
tags: ["ACM"]
comments : true
---

# 硬币找钱问题

## Description

设有6 种不同面值的硬币，各硬币的面值分别为5 分，1 角，2 角，5 角，1 元，2元。现要用这些面值的硬币来购物和找钱。购物时可以使用的各种面值的硬币个数存于数组Coins[1:6]中，商店里各面值的硬币有足够多。在1次购物中希望使用最少硬币个数。例如，1 次购物需要付款0.55 元，没有5 角的硬币，只好用2*20+10+5 共4 枚硬币来付款。如果付出1 元，找回4 角5 分，同样需要4 枚硬币。但是如果付出1.05 元（1 枚1元和1 枚5分），找回5 角，只需要3 枚硬币。这个方案用的硬币个数最少。

对于给定的各种面值的硬币个数和付款金额，计算使用硬币个数最少的交易方案。

## Input

输入数据有若干组，每一行有6 个整数和1 个有2 位小数的实数。分别表示可以使用的各种面值的硬币个数和付款金额。文件以6 个0 结束。

## Output

将计算出的最少硬币个数输出。结果应分行输出，每行一个数据。如果不可能完成交易，则输出“impossible”。

## Sample Input

```
2 4 2 2 1 0 0.95
2 4 2 0 1 0 0.55
0 0 0 0 0 0
``` 

## Sample Output

```
2
3
``` 

### Source ：《算法设计与分析》

## 解题思路

01背包，完全背包
change[i]表示商店支付面值为i需要的最少硬币个数；
dp[i]表示顾客现有的硬币数支付面值为i需要的最少硬币数；
w为当前要支付的实际面值，若顾客支付面值为k的钱（k>=w）,商家找钱k-w,该条件下最少需要的硬币数为dp[k]+change[k-w],
由此推得，最少硬币数为所有符合条件k>=w下最小的dp[k]+change[k-w];
即： ans = min(dp[k]+change[k-w])(k>=w)

对于change[i],商店里各面值的硬币有足够多，故可用完全背包实现
对于dp[i],可用混合背包计算，这里我直接拆成01背包来实现（比较暴力，O(∩_∩)O~）。
PS:为减少空间开销，最终化为以5分为单位计算

    其实，这个算法在时间和空间上的牺牲还是比较大的，可用贪心进行优化，可惜当年没深入去想……

Answer

```

#include<stdio.h>
#include<string.h>

constint N =20000;
int change[N];//change[i]为面值为i的钱至少需要的硬币个数
int dp[N];//dp[i]为当前拥有的硬币数量条件下表示面值为i的最少硬币个数
int value[6] = {1,2,4,10,20,40};//每种硬币对应面值，依次为1，2,4,10,20,40个五分，即5,10,20,50,100,200；
int number[6];//对应于当前拥有的每种硬币个数

void init()//计算change[i]
{
   int i,j;
   for(i=0;i<N;i++)change[i]=-1;
   change[0]=0;
   for(i=0;i<6;i++)
   {
      for(j=value[i];j<N;j++)//这里使用完全背包，不能理解的话可参考背包九讲
      {
       if(change[j-value[i]]!=-1)
       {
         int temp=change[j-value[i]]+1;
         if(change[j]==-1||temp<change[j])change[j]=temp;
       }
      }
   }
}
int main()
{
   //freopen("change.in","r",stdin);
   
    init(); //计算出change[i]
 
    while(scanf("%d%d%d%d%d%d",&number[0],&number[1],&number[2],&number[3],&number[4],&number[5])!=EOF)
    {
      int sum =0;
      int kk;
      for(kk=0;kk<6;kk++)sum+=number[kk];
      if(sum==0)break;
      double weight;
      scanf("%lf",&weight);
      weight=weight*100;
     // printf("weight = %lf\n",weight);
int w =int(weight+0.0000001);//处理精度问题
      //printf("%d\n",w);

      if(w%5!=0)//若不能整除，则无法表示
      {
         printf("impossible\n");
         continue;
      }
      else
          w = w/5;
     
      int i,j;
      memset(dp,-1,sizeof(dp));
      dp[0]=0;
      int bigger =0;
      for(i=0;i<6;i++)//计算顾客支付面值i需要的最少硬币数dp[i]
      {
        while(number[i]--) //将混合背包拆成01背包做，写水了点。。。
        {
         bigger = bigger+value[i];
         for(j=bigger;j>=value[i];j--)
         {
          if(dp[j-value[i]]!=-1)
          {
            int temp=dp[j-value[i]]+1;
            if(dp[j]==-1||temp<dp[j])
            {
              dp[j]=temp;
            }
          }
         }
        }
      }
 
    int ans =-1;
    for(i=w;i<=bigger;i++)//寻找最少硬币组合
    {
     if(dp[i]!=-1)
     {
      int need = i-w;
      if(change[need]!=-1)
      {
       int temp = dp[i]+change[need];
       if(ans==-1||ans>temp)ans=temp;
      }
     }
    }

   // for(i=0;i<N;i++)
  //   if(dp[i]!=-1)
   //  printf("dp[%d]=%d\n",i,dp[i]);

    if(ans!=-1)
    printf("%d\n",ans);
    else
     printf("impossible\n");
   }
   return0;
}

```


