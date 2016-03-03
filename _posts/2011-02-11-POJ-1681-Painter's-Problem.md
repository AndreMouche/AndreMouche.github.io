---
layout: post
title: "POJ 1681 Painter's Problem【状态压缩，枚举】"
keywords: ["algorithm", "POJ"]
description: "POJ 1681 Painter's Problem 解题报告"
category: "algorithm"
tags: ["ACM","dp"]

---

[POJ 1681 Painter's Problem](http://poj.org/problem?id=1681)
#算法核心：
 状态压缩，枚举
#大意：
有一面n*n的墙，对其中某一格子上色，
则其上、下、左、右及自身的颜色均变色，
颜色仅有黄色和白色两种，已知墙面信息，
问能否将墙面全部变为黄色，若能，至少需要涂色几次？

#分析：
1. 通过状态压缩，枚举第一行的着色网格
2. 通过已知的第一行着色状态，根据上层信息依次推得下层着色状态
3. 对2退出的第n层着色状态进行判断，若合法，更新当前最小值。

#Answer

```
#include<stdio.h>
#include<string.h>
constint N =17;
constint inf =99999;
bool graph[N][N];//graph[i][j]存储最初的颜色，黄色为0，白色为1
bool swap[N][N];//swap[i][j]存储该格是否进行染色
int main()
{
 int T;
 while(scanf("%d",&T)!=EOF)
 {
  while(T--)
  {
   int n;
   scanf("%d",&n);
   long upper =1<<n;
   long cur,i,j;
   char str[N];
   memset(graph,0,sizeof(graph));
   for(i=1;i<=n;i++)
   {
    scanf("%s",str);
    for(j=0;j<n;j++)
    {
                  if(str[j]=='w')graph[i][j+1]=true;
    }
   }
   int ans = inf;
   for(cur=0;cur<upper;cur++)//枚举第一行按钮状态
   {
    memset(swap,false,sizeof(swap));
    long tcur = cur;
    for(i=1;i<=n;i++)
    {
     swap[1][i]=cur&1;
     cur>>=1;
    }
               cur=tcur;
    for(i=2;i<=n;i++)
    {
                      for(j=1;j<=n;j++)
        swap[i][j]=(graph[i-1][j]^swap[i-1][j]^swap[i-1][j-1]^swap[i-1][j+1]^swap[i-2][j]);
    }

    bool flag =true;
       
    for(i=1;i<=n;i++)
     if(swap[n][i]^swap[n-1][i]^swap[n][i-1]^swap[n][i+1]^graph[n][i]){
     flag=false;
     break;
     }

    if(flag)
    {
                   int step=0;
       for(i=1;i<=n;i++)
        for(j=1;j<=n;j++)
         if(swap[i][j])
         step++;

        if(step<ans)ans=step;
    }
   }

            if(ans==inf)printf("inf\n");
   else
    printf("%d\n",ans);
  }
 
 }
 return0;
}
```