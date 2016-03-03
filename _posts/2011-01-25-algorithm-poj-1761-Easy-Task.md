---
layout: post
title: "POJ 1761 Easy Task"
keywords: ["algorithm", "POJ"]
description: "POJ 1761 Easy Task 解题报告"
category: "algorithm"
tags: ["ACM"]

---
[POJ 1761 Easy Task](http://poj.org/problem?id=1761)
##题意
 给你n条提交问题的信息，每条信息包含提交时间，提交队伍，提交题目编号，是否AC信息，统计这n条信息，要求输出每个问题的信息，包括题号，提交次数，平均提交次数，平均提交时间
 
PS:

1.某一队伍一旦AC了某一道题后，过后再提交该题的信息不计入统计

2.只对已经AC的队伍进行统计，即提交次数 = SUM(已AC的队伍的总共提交次数)，不对未AC的队伍进行统计

##分析：
  1. 用map<string,int>存储队伍信息，编号
  2. 用accept[i][j]表示第j支队伍是否AC问题i
  3. 用actime[i][j]表示第j支队伍共提交问题i的次数
  4. 用node保存一个问题的信息，包括解决该问题的队伍数，总时间。

  在计算某个问题的总共提交次数时，抛弃为AC的队伍，
  亦对于问题i,只有当队伍j已经AC了该题目时（accept[i][j]==true）,
  才将其提交次数actime[i][j]统计入i题的总提交次数内

##Answer

```c++
#include<stdio.h>
#include<string>
#include<iostream>
#include<map>
usingnamespace std;
constint N =9;//问题数目
constint M =100;//队伍数目
struct node
{
    double totalCost;//所需要消耗的总时间
int acTeam;//解决问题的队伍数目
int submit;//提交次数
    
}problem[N];//包含每个问题的基本信息
 
bool accept[N][M];//accept[i][j]=true表示第j个队伍已解决问题i
int actime[N][M];//actime[i][j]表示第j支队伍已提交问题i的次数
int main()
{
    int n;
    char info[100];
    map<string,int>team;//team[队名] = 编号
    map<string,int>::iterator iter;
    memset(actime,0,sizeof(actime));
    while(scanf("%d",&n)!=EOF)
    {
        team.clear();
        double cost;
        string teamInfo,problemInfo,acInfo;
        int id =0;
        int i;
        for(i=0;i<N;i++)
        {
            problem[i].acTeam=0;
        
            problem[i].submit=0;
            problem[i].totalCost =0;
        }

        memset(accept,false,sizeof(accept));
        memset(actime,0,sizeof(actime));
        while(n--)
        {
           //scanf("%d",&cost);
           cin>>cost>>teamInfo>>problemInfo>>acInfo;
          // cout<<cost<<" "<<teamInfo<<" "<<problemInfo<<" "<<acInfo<<endl;
           iter = team.find(teamInfo);
            int teamId;
           if(iter == team.end())
           {
               team[teamInfo] = id;
               teamId = id;
               id++;
           }
           else
               teamId = iter->second;
          
           int Qid = problemInfo[0]-'A';//问题编号
if(accept[Qid][teamId])continue;//该队伍已解决当前问题
           actime[Qid][teamId]++;
           
           if(acInfo[0]=='A')
           {
               problem[Qid].acTeam++;
               problem[Qid].totalCost+=cost;
               accept[Qid][teamId]=true;
           }
           
           //cout<<teamInfo<<endl;
           
        }

     for(i=0;i<N;i++)
     {
         printf("%c %d",i+'A',problem[i].acTeam);
         if(problem[i].acTeam ==0)
         {
             printf("\n");
             continue;
         }

         int j =0;
         problem[i].submit =0;
         for(j=0;j<M;j++)
         {
             if(accept[i][j])
                 problem[i].submit+=actime[i][j];
         }
         printf(" %.2lf %.2lf\n",problem[i].submit*1.0/problem[i].acTeam,problem[i].totalCost/problem[i].acTeam);
     }

    }
    return0;
}
```