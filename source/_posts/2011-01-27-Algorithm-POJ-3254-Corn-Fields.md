---
layout: post
title: "POJ 3254 Corn Fields【dp 状态压缩】"
keywords: ["algorithm", "POJ"]
description: "POJ 3254 Corn Fields【dp 状态压缩】解题报告"
category: "algorithm"
tags: ["ACM"]
date: 2011-01-27
comments: true 
---

[POJ 3254  Corn Fields](http://poj.org/problem?id=3254)

# 算法核心

状态压缩,DP

# 题意

输入m行n列的数字，其中为1或者是0

1表示土壤肥沃可以种植草地，0则不可以。

在种草地的区域可以放牛，但相邻的两
块区域不允许同时放牛，问有多少种放牛的方法？
（不放牛也算一种情况）

# 分析
 由m,n<=12,可用状态压缩
 
 对于第i行，可以放草的格子置为0，不可以种草的格子设置为1，整一行的状态存入graph[i]中
 
 对于每一行，放牛的格为1，不放牛的格为0，整行用一个二进制数表示
 dp[i][j]表示第i行放牛状态为j时有多少种方法，易知：
 
 1. 首先j必须合法，即左右相邻两位不同时出现1，
 2. 不能在不能种草的地方放牛，即j&graph[i]==0
 3. dp[i][j] = SUM(dp[i-1][k]),其中k&j==0,即上下相邻位置不放牛

由此，可以求出所有的dp[i][j]，那么放牛的种类共有 = SUM(dp[n-1][j])最后一行所有状态的放牛种类之和

```
#include<stdio.h>
#include<string.h>
constint N =1<<14;
bool legal[N];//legal表示单行出现该状态时是否合法
int leg[N];//统计单行合法数据
int legNum;//合法状态的个数
long dp[13][N];//dp[i][j]表示第i行放牛状态为j时有多少种方法
long graph[13];//记录每行状态，不能种草的为1，能种的为0

void getLeg()//获取合法单行的状态
{
 int i;
    legNum =0;
 memset(legal,true,sizeof(legal));
 leg[legNum++]=0;
 for(i=1;i<N;i++)
 {
       int temp = i;
    while(temp)
    {
     int curt = temp;
     temp>>=1;
     if(legal[temp]==false)
     {
      legal[i]=false;break;
     }

           if((temp&1)&&((curt&1)))
     {
      legal[i]=false;
      break;
     }
    }
    if(legal[i])
     leg[legNum++] = i;
 }
}

int getId(int m)//二分获取每行为m格时合法状态的上限
{
 int left =0;
 int ans =0;
 int right = legNum;
 while(left<=right)
 {
  int mid = (left+right)>>1;
  if(leg[mid]>m)
  {
               ans = mid;
      right = mid-1;
  }
  else
   left=mid+1;
 }
 return ans;
}

int main()
{
    getLeg();
 int n,m;
 while(scanf("%d%d",&n,&m)!=EOF)
 {
        int i,j,d;
  for(i=0;i<n;i++)
  {
   graph[i]=0;
   for(j=0;j<m;j++)
   {
    scanf("%d",&d);  //将状态取反
    d =1-d;  
    graph[i]=graph[i]*2+d;
   }
  }
        int upper = getId((1<<m)-1);//取得m个数的合法上限
  memset(dp,0,sizeof(dp));
 // printf("%d %d\n",upper,leg[upper-1]);
  //设置第一层
for(i=0;i<upper;i++)
  {
   if((leg[i]&graph[0])==0)//排除放在废弃区域
   dp[0][leg[i]] =1;
  }
  
  //设置余下空间

  for(i=1;i<n;i++)
  {
   for(d=0;d<upper;d++)
   { 
    int pre = leg[d];   
    if(dp[i-1][pre]>0)//剪枝
for(j=0;j<upper;j++)
    { 
     int cur = leg[j];
     if((cur&graph[i])==0)//没有在废弃区种草
     {
     
      if((pre&cur)==0)//上下没有相邻
      {
       dp[i][cur]+=dp[i-1][pre];
      }
     }
    }
   }
  }

  long ans =0;
  for(i=0;i<upper;i++)
  {
   ans+=dp[n-1][leg[i]];
   ans %=100000000;
  }

  printf("%ld\n",ans);
 }
 
 return0;
}
```
