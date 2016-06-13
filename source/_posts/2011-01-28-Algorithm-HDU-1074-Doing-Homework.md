---
layout: post
title: "HDU 1074 Doing Homework [状态压缩DP]"
keywords: ["algorithm", "POJ"]
description: "HDU 1074 Doing Homework 解题报告"
category: "algorithm"
tags: ["ACM","DP"]
date: 2011-01-28
comments: true
---
[HDU 1074 Doing Homework ](http://acm.hdu.edu.cn/showproblem.php?pid=1074)

# 算法核心：

状态压缩DP

# 大意：

有n门课程作业，每门作业的截止时间为D,需要花费的时间为C，若作业不能按时完成，每超期1天扣1分。
这n门作业按课程的字典序先后输入
问完成这n门作业至少要扣多少分，并输出扣分最少的做作业顺序

PS:达到扣分最少的方案有多种，请输出字典序最小的那一组方案

# 分析：

n<=15，由题意知，只需对这n份作业进行全排列，选出扣分最少的即可。
用一个二进制数存储这n份作业的完成情况，第1.。。。n个作业状况分别
对应二进制数的第0，1.。。。。,n-1位则由题意，故数字上限为2^n
其中 2^n-1即为n项作业全部完成，0为没有作业完成。。。

用dp[i]记录完成作业状态为i时的信息（所需时间，前一个状态，最少损失的分数）。
递推条件如下

1. 状态a能做第i号作业的条件是a中作业i尚未完成，即a&i=0。
2. 若有两个状态dp[a],dp[b]都能到达dp[i],那么选择能使到达i扣分小的那一条路径，若分数相同，转入3
3. 这两种状态扣的分数相同，那么选择字典序小的，由于作业按字典序输入，故即dp[i].pre = min(a,b);

初始化：dp[0].cost = 0;dp[0].pre=-1;dp[0].reduced = 0;

最后dp[2^n-1].reduced即为最少扣分，课程安排可递归的输出

# Answer

```

#include<stdio.h>
#include<string.h>
constint N =65536;

struct node
{ 
 int cost;//所需要的时间
int pre;//前一状态
int reduced;//最少损失的分数
}dp[N];//dp[i][j]表示在第i天完成作业信息为j
bool visited[N];//表示完成j的状态是否被访问

struct course
{
 int deadtime;//截止日期
int cost;//所需日期
char name[201];
}course[16];

void output(int status)//递归输出课程安排表
{
 int curjob= dp[status].pre^status;
 int curid =0;
 curjob>>=1;
 while(curjob)
 {
  curid++;
  curjob>>=1;
 }
 if(dp[status].pre!=0)//输出其前面的课程
 {
  int preday = dp[status].cost-course[curid].cost;
  output(dp[status].pre);
 }
 printf("%s\n",course[curid].name);
}


int main()
{
 int T;
 while(scanf("%d",&T)!=EOF)
  while(T--)
 {
  int n;
  scanf("%d",&n);//n门课程
int i,j;
  int upper =1<<(n);
        int dayupper =0;
  for(i=0;i<n;i++)
  {
   scanf("%s%d%d",course[i].name,&course[i].deadtime,&course[i].cost);
      dayupper+=course[i].cost;
  }

      memset(visited,false,sizeof(visited));

   dp[0].cost=0;
   dp[0].pre=-1;
   dp[0].reduced =0;
   visited[0]=true;

   int work;
   int tupper = upper-1;

   for(j=0;j<tupper;j++)//遍历所有状态
   {
    
     for(work=0;work<n;work++)
     {
      int cur =1<<work;
      if((cur&j)==0)//该项作业尚未做过
      {
                   int curtemp=cur|j; 
       int day=dp[j].cost+course[work].cost;
       dp[curtemp].cost = day;
                   int reduce = day-course[work].deadtime;
       if(reduce<0)reduce=0;
       reduce+=dp[j].reduced;
      
       if(visited[curtemp])//该状态已有访问信息
       {
        if(reduce<dp[curtemp].reduced)
        {
         dp[curtemp].reduced=reduce;
         dp[curtemp].pre=j;
        }
        //else
        // if(reduce==dp[curtemp].reduced)//扣分相同，取字典序小的那一个，由于这
        // {                                 //里j是按从小到达搜索的，默认已是按字典序，不需再处理
        //  if(dp[curtemp].pre>j)
        //   dp[curtemp].pre = j;
        // }
       }
       else
        if(visited[curtemp]==false)//该状态尚未到达过
        {
         visited[curtemp]=true;
                           dp[curtemp].reduced=reduce;
         dp[curtemp].pre=j;
        }
      }
     }
   }
  
   printf("%d\n",dp[tupper].reduced);  
   output(tupper);
   
 }
 return0;
}
```

