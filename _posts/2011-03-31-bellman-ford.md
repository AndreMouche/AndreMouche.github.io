---
layout: postlayout
title: POJ 3259 Wormholes【Bellman-Ford】
thumbimg: 1346208288725.jpg
categories: [acm]
tags: [acm, graph, algorithm]
---


[POJ 3259 Wormholes](http://poj.org/problem?id=3259)
大意：给出由n个顶点组成的有向图所有边信息，判断该图中是否存在负回路
分析：
  由于不知道图中的负环路会包含哪些顶点，故自定义一源点s,对于每个顶点k,
有一条s到k权值为0的有向边，此时便可初始化s到每个顶点的距离dis[k]为0，
然后用Bellman-Ford计算从源点s到各点的単源最短路，若存在某条边有第n+1次
松弛操作，则存在负环，否则没有负环

重点注意：引出源点s的定义，并创建从s到每个点的单向边的目的是为更好地理解dis[i]的初值问题，
当然，对于初值dis[]不一定要是0，也可以是其他任意数，
但理解的前提是创建从s到每个点的边必须是单向边且只能是单向。

{% highlight c %}
#include<stdio.h>
#include<string.h>
 constint N =2500+200+5;
 constint MAX_N =500+5;
 struct Edge
{
int from;
int to;
int cost;
}edge[N*2];
int ednum;
int dis[MAX_N];//dis[i]记录从源点s到i的当前距离
//判断图中是否存在负权回路，若不存在返回true
bool bellman(int n)
{
int i,j;
    memset(dis,0,sizeof(dis));//自定义一源点s,从s到每个顶点存在单向边，距离为0

for(i=0;i<n;i++)//对每条边进行n次松弛操作
 {
bool changed =false;
for(j=0;j<ednum;j++)
  {
int temp = dis[edge[j].from]+edge[j].cost;
if(temp<dis[edge[j].to])
   {
    dis[edge[j].to]=temp;
    changed =true;
   }
  }

if(changed==false)returntrue;
 }

for(j=0;j<ednum;j++)//进行第n+1次松弛操作，判断有无负回路
 {
if(dis[edge[j].from]+edge[j].cost<dis[edge[j].to])
returnfalse;
 }
returntrue;
}

int main()
{
int F;
while(scanf("%d",&F)!=EOF)//农场个数
 {
while(F--)
   {
int n,m,w,i;
   scanf("%d%d%d",&n,&m,&w);//n个点，m条无向边，w条有向负边
   ednum =0;
for(i=0;i<m;i++)
   {
    scanf("%d%d%d",&edge[ednum].from,&edge[ednum].to,&edge[ednum].cost);
    ednum++;
    edge[ednum].to=edge[ednum-1].from;
    edge[ednum].from=edge[ednum-1].to;
    edge[ednum].cost=edge[ednum-1].cost;
    ednum++;
   }

for(i=0;i<w;i++)
   {
    scanf("%d%d%d",&edge[ednum].from,&edge[ednum].to,&edge[ednum].cost);
    edge[ednum].cost =-edge[ednum].cost;
    ednum++;
   }

if(bellman(n)==false)
   {
    printf("YES\n");
   }
else
    printf("NO\n");
   
   }
 }
return0;
}

{% endhighlight %}


