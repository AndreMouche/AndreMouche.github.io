---
layout: post
title: "POJ 1806 Manhattan 2025"
keywords: ["algorithm", "POJ"]
description: "POJ 1806 Manhattan 2025 解题报告"
category: "algorithm"
tags: ["ACM","DP"]
date: 2011-01-27
comments: true
---
[POJ 1806 Manhattan 2025 ](http://poj.org/problem?id=1806)

# 大意

在一个三维空间里面，有一交通工具通过一单位长度需要一升汽油，现有n升汽油，画出该交通工具在各层的运输情况
      
将每一层简化为一个以交通工具所在位置为中心的二维网格图，在可达网格内写入到达该网格所需要的汽油数。

   自底向上画出每一层所在的二维图。

   当n>9时，不需要统计

# Example:

 n = 2 时，若标记当前这一层为0层，则该情况下交通工具所能达到的层次为-2层到2层，即共5层，分别为-2,-1,0,1,2层,将每一层的二维图输出即可。
 
题目中要求将最底层即为1，那么在该情况下，上述各层对应为第1,2,3,4,5层，其中交通工
具所在的位置为第3层。

# 分析
 通过简单的推断可发现，各层的情况以当前交通工具所在层为中心对称，故可用递归实现~

 
# Answer

```
#include<stdio.h>
#include<math.h>
void draw(int n,int row,int floor)//画出距离交通工具所在层floor层的二维图
{
    int i,j;
    for(i=0;i<row;i++)
    {
        int needi = abs(i-n);//在垂直方向需要的步骤
for(j=0;j<row;j++)
        {
            int needj = abs(j-n);//水平方向需要的步骤
int need = needi+needj+floor;
            if(need<=n)printf("%d",need);
            else
                printf(".");
        }
        printf("\n");
    }
}
void output(int n,int row,int floor)//共有n汽油，有row行，现在第floor层
{
    printf("slice #%d:\n",n-floor+1);
    if(n>9)//若汽油大于9
    {
        draw(n,row,floor+1);
        return;
    }
    draw(n,row,floor);//画下面的第floor层
if(floor==0)return;
    output(n,row,floor-1);//画中间的那一部分
    printf("slice #%d:\n",n+floor+1);
    draw(n,row,floor);//上面的第floor层


}

int main()
{
    int T;
    while(scanf("%d",&T)!=EOF)
    {
        int cases;
        for(cases=1;cases<=T;cases++)
        {
            int n;
            scanf("%d",&n);
            printf("Scenario #%d:\n",cases);
            output(n,2*n+1,n);
            printf("\n");

        }
    }
    return0;
}
```
