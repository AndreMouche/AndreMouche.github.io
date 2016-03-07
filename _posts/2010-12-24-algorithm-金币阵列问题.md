---
layout: post
title: "金币阵列问题"
keywords: ["algorithm", "算法设计与分析"]
description: "金币阵列问题解题报告"
category: "algorithm"
tags: ["ACM"]
comments: true 
---

# 金币阵列问题

## Description

有m*n(m <= 100,n <= 100)个金币在桌面上排成一个m行n 列的金币阵列。每一枚金币或正面朝上或背面朝上。用数字表示金币状态，0表示金币正面朝上，1 表示背面朝上。
金币阵列游戏的规则是：
（1）每次可将任一行金币翻过来放在原来的位置上；
（2）每次可任选2 列，交换这2 列金币的位置。
算法设计：
给定金币阵列的初始状态和目标状态，计算按金币游戏规则，将金币阵列从初始状态变
换到目标状态所需的最少变换次数。


##数据输入：
文件中有多组数据。文件的第1行有1 个正整数k，表
示有k 组数据。每组数据的第1 行有2 个正整数m 和n。以下的m行是金币阵列的初始状
态，每行有n 个数字表示该行金币的状态，0 表示金币正面朝上，1 表示背面朝上。接着的
m行是金币阵列的目标状态。

##结果输出:
相应数据无解时
输出-1。

输入文件示例

```
2
4 3
1 0 1
0 0 0
1 1 0
1 0 1
1 0 1
1 1 1
0 1 1
1 0 1
4 3
1 0 1
0 0 0
1 0 0
1 1 1
1 1 0
1 1 1
0 1 1
1 0 1
```

输入文件示例

```
2
-1
```

Source:《算法设计与分析习题解答》

PS:解题思路与课本类似，但课本答案有误。

## Answer

```
代码

#include<stdio.h>
constint inf =99999;
constint N =101;

int a[N][N],b[N][N],temp[N][N]; //a存储初始矩阵，b为目标状态矩阵
int n,m;
int need;//需要变换次数
void ChangeL(int x,int y)//变换列
{
    if(x==y)return;
    int i;
    for(i=1;i<=n;i++)
    {
        int tt=temp[i][y];
        temp[i][y]=temp[i][x];
        temp[i][x]=tt;
    }
    need++;
}
void ChangeH(int x)//变换行
{
    int i;
    for(i=1;i<=m;i++)
    {
      temp[x][i]^=1;
    }
}

bool Same(int x,int y) //判断列是否满足条件
{
   int i;
   for(i=1;i<=n;i++)
       if(b[i][x]!=temp[i][y])returnfalse;
       returntrue;
}

int main()
{
    int tests;
    scanf("%d",&tests); //数据组数
while(tests--)
    {
        scanf("%d%d",&n,&m); //n行，m列
int i,j;
        for(i=1;i<=n;i++)
            for(j=1;j<=m;j++)
            {
                scanf("%d",&a[i][j]);
            }
             

        for(i=1;i<=n;i++)
            for(j=1;j<=m;j++)
                scanf("%d",&b[i][j]);

        int k;
        int ans=inf; //ans存储最终答案，初始值为无穷大

        for(k=1;k<=m;k++)//枚举各列为第一列
        {
          for(i=1;i<=n;i++)
              for(j=1;j<=m;j++)
                  temp[i][j]=a[i][j];
          need=0;
          ChangeL(1,k);
          for(i=1;i<=n;i++)
          {
              if(temp[i][1]!=b[i][1])//该行不满足条件
              {
                  ChangeH(i);//变换行
                  need++;
              }
          }

          bool find;
          for(i=1;i<=m;i++)//检查每列是否满足条件
          {
              find=false;
              if(Same(i,i))
              {
                  find=true;continue;
              }
              for(j=i+1;j<=m;j++)//寻找temp中与b的i列相同的列
              {
                 if(Same(i,j))
                 {
                     if(Same(j,j))continue;
                     ChangeL(i,j);
                     find=true;
                     break;
                 }
              }
              if(find==false)//找不到该列对应列
              {
                break;
              }
          }

         if(find==true&&need<ans)ans=need;
        }

        if(ans<inf)printf("%d\n",ans);
        else
            printf("-1\n");
    }
    return0;
}
```
