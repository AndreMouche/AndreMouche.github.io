---
layout: post
title: "POJ 3067 Japan【一维树状数组】"
keywords: ["algorithm", "POJ"]
description: "POJ 3067 Japan 解题报告"
category: "algorithm"
tags: ["ACM","dp"]

---

POJ 3067 Japan【一维树状数组】
=========

[POJ 3067 Japan](http://poj.org/problem?id=3067)

核心算法：
--------
一维树状数组
大意
------
Japan在东边有n座城市，从北到南编号依次为1,2,3...n
在西边有m座城市，从北到南编号分别为1,2,3...m
现要在南北城市之间修建k条超级高速公路，求会出现多少个十字路口
（注：每个十字路口只能由两条交叉的路所构成）
输入：
-------
```
T   .....cases数
n,m,k
以下k行每行对应于一条高速公路，由两个数字（xi,yi）组成，xi对应于东城编号,yi西城编号
```
输出：
-----
```
十字路口数目
```

分析：

1. 对该数组先按y从大到小排序，若y相等，则按x从大到小排序
2. 从前往后扫描各条高速公路，对路（xi,yi）其与前面点的交点数目为其左上角路的个数，即所有的(xj,yj),其中xj<xi,yj>yi.
3. 与poj 2352类似地，用数组数组c[i]存储x轴的信息

Answer
---
```
#include<stdio.h>
#include<string.h>
#include<algorithm>
using namespace std;
const int N = 1001;
const int K = 1000010;

__int64 c[N];
int n,m,k;

struct node
{
 int x,y;
 bool operator <(const node &A){//先按y从大到小排序，若y相同，按x从小到大排序
 if(y==A.y)return x>A.x;
 return y>=A.y;
 }
}point[K];
inline int lowbit(int x)
{
 return x&(-x);
}

void modify(int x,int add)
{
   while(x<=n)
   {
    c[x]+=add;
    x+=lowbit(x);
   }
}
__int64 sum(int x)
{
 __int64 ans=0;
 while(x>0)
 {
  ans+=c[x];
  x-=lowbit(x);
 }
 return ans;
}
int main()
{
 int T;
 while(scanf("%d",&T)!=EOF)
 {
  int cases;
  for(cases=1;cases<=T;cases++)
  {
   memset(c,0,sizeof(c));
  
   scanf("%d%d%d",&n,&m,&k);
   int i;
   __int64 pnum = 0;//点个数
   for(i=0;i<k;i++)
   {
    int xx,yy;
    scanf("%d%d",&xx,&yy);
    
    {
    
     point[pnum].x = xx;
     point[pnum].y = yy;
     pnum++;
    }
   }

   sort(point,point+pnum);//对其排序

   __int64 ans = 0;
   for(i=0;i<pnum;i++)
   {
      modify(point[i].x,1);
      ans+=sum(point[i].x-1);
   }
           printf("Test case %d: %I64d\n",cases,ans);
  }
 }
 return 0;
}
```