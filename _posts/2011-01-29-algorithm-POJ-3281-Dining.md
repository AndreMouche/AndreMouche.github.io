---
layout: post
title: "POJ 3281 Dining【Dinic】"
keywords: ["algorithm", "POJ"]
description: "POJ 3281 Dining 解题报告"
category: "algorithm"
tags: ["ACM","网络流"]

---
[POJ 3281 Dining【Dinic】 ](http://poj.org/problem?id=3281)

## 核心算法
网络最大流

##大意：
 
* 有n头牛，F种食物，D种饮料，
 * 第i头牛喜欢fi种食物，di种饮料，编号分别为。。。
 
  已知一头牛最多能吃一种食物和一种饮料，每种饮料
  或食物最多能被一头牛吃，求以上条件下，最多能有多少头
  牛能吃到他所喜爱的食物和饮料

##建立模型：
  建立网络流模型：
   
1. 对每种食物建立从源点指向它的一条边，流量为1
2. 在牛与它所喜爱的食物间建立一条边，流量为1
3. 在牛与它所喜欢的饮料间建立一条边，流量为1
4. 对每种饮料建立一条指向汇点的边，流量为1
5. 在上面的基础上，将牛拆点，在拆开的点间建立一条流量为1的边
   在以上条件下，从源点到汇点的最大流即为答案

##模型的分析：
  
* 条件1使得满足每种食物有且只有一个，条件4类似
* 条件2使得每头牛只能选择自己喜欢的食物，条件3类似
* 条件5使得每头牛最多只能选择一种饮料和食物

这里  最大流使用的是dinic算法。。

##Code
```
#include<stdio.h>
#include<string.h>
constint EDGE_NUM =20001;//边数
constint POINT_NUM =501;//点数
struct edge
{
 int v;//点
int next;//下一边
int value;//当前边流量
}edge[2*EDGE_NUM];//边信息，以邻接表形式存储
int p[POINT_NUM];//p[i]记录最后一条以i为起点的边的id,即以i为起点的最后一条边为edge[p[i]],而edge[p[i]].next则为以i为起点的倒数第二条边，以此类推
int level[POINT_NUM];//level[i]记录i点的层次
int que[POINT_NUM],out[POINT_NUM];//辅助数组
int edgeNumber;
void init()
{
 edgeNumber =0;
 memset(p,-1,sizeof(p));
}
inline void addEdge(int from,int to,int value)//添加边，以邻接表形式存储
{
 edge[edgeNumber].v = to;
 edge[edgeNumber].value = value;
 edge[edgeNumber].next = p[from];
 p[from] = edgeNumber++;

}

int Dinic(int source,int sink,int n)
{
 int i,maxFlow =0;
 while(true)
 {
  int head,tail;
  for(i=0;i<n;i++)level[i] =0;
  level[source] =1;//源点为第一层
  head =0;tail =0;
  que[0] = source;//que这里当队里使用
while(head<=tail)//BFS该剩余图，计算每个可达点层次
  {
   int cur = que[head++];
   for(i=p[cur];i!=-1;i=edge[i].next)
   {
    if(edge[i].value>0&&level[edge[i].v]==0)
    {
     level[edge[i].v] = level[cur]+1;
     que[++tail] = edge[i].v;
    }
   }
  }


  if(level[sink]==0)break;//不存在增广路

  for(i=0;i<n;i++)out[i]=p[i];//out[i]动态记录可用边

  int q =-1;//q为已经搜索到的点的个数,que存放途径边信息
while(true)//DFS剩余图，查找增广路
  {
   if(q<0)//当前路为空
   {
               int cur =out[source];
      for(;cur!=-1;cur=edge[cur].next)//查找第一条边
      {
                   if(edge[cur].value>0&&out[edge[cur].v]!=-1&&level[edge[cur].v]==2)//合法第一条边必须满足：1.流量大于0;2.终点有可用边 3:终点层次为2
break;
      }

      if(cur==-1)break;//找不到第二层，当前剩余图已经没有增广路

      que[++q]=cur;//存入第一条边id
out[source]=edge[cur].next;
   }

   int curnode = edge[que[q]].v;//当前路的终点
   
   if(curnode==sink)//找到一条增广路
   {
                int thisflow = edge[que[0]].value;//thisflow为当前增广路的流量
int index =0;//标记最小流量边的id
for(i=1;i<=q;i++)
    {
     if(thisflow>edge[que[i]].value)
     {
      thisflow=edge[que[i]].value;
      index = i;
     }
    }

    maxFlow+=thisflow;
    for(i=0;i<=q;i++)
    {
     edge[que[i]].value-=thisflow;
     edge[que[i]^1].value+=thisflow;//与其方向相反的边
    }
              
    q = index-1;//查找下一条增广路时可直接使用当前路的前q条边
 
   }
   else//尚未找到汇点
   {
    int cur =out[curnode];
    for(;cur!=-1;cur=edge[cur].next)
    {
     if(edge[cur].value>0&&out[edge[cur].v]!=-1&&level[edge[cur].v]==level[curnode]+1)
      break;
    }
    if(cur==-1)//没有下一条路
    {
     out[curnode]=-1;//标记当前点的可达边为0
     q--;
    }
    else
    {
     que[++q]=cur;
     out[curnode]=edge[cur].next;//下一次搜索时可达边从edge[cur].next开始查找
    }
   }
  }

 

 }

 return maxFlow;

}
int main()
{
   int Nn,Ff,Dd;
   while(scanf("%d%d%d",&Nn,&Ff,&Dd)!=EOF)
   {
      init();
      int foodstart =1;
      int cow1 = Ff+2;
      int cow2 = cow1+Nn+1;
      int drinkstart = cow2+Nn+1;
      int end = drinkstart+Dd+1;

      int i;
      for(i=0;i<Nn;i++)//添加牛边
      {
           addEdge(cow1+i,cow2+i,1);
           addEdge(cow2+i,cow1+i,0);
      }

      for(i=0;i<Ff;i++)//添加食物边
      {
          addEdge(0,foodstart+i,1);
          addEdge(foodstart+i,0,0);
      }

      for(i=0;i<Dd;i++)//添加饮料
      {
          addEdge(drinkstart+i,end,1);
          addEdge(end,drinkstart+i,0);
      }

      for(i=0;i<Nn;i++)
      {
          int f,d;
          scanf("%d%d",&f,&d);
          int x;
          while(f--)
          {
             scanf("%d",&x);
             x--;
             addEdge(foodstart+x,cow1+i,1);
             addEdge(cow1+i,foodstart+x,0);
          }
          while(d--)
          {
              scanf("%d",&x);
              x--;
              addEdge(cow2+i,drinkstart+x,1);
              addEdge(drinkstart+x,cow2+i,0);
          }
      }


      printf("%d\n",Dinic(0,end,end+1));
   }
   return0;
}
```

