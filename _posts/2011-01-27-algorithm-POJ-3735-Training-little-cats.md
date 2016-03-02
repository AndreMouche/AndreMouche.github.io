---
layout: post
title: "POJ 3735 Training little cats【矩阵的快速求幂】"
keywords: ["algorithm", "POJ"]
description: "POJ 3735 Training little cats 解题报告"
category: "algorithm"
tags: ["ACM"]

---

[POJ 3735 Training little cats](http://poj.org/problem?id=3735)
#算法核心：
矩阵建模，矩阵的快速幂


#大意：
已知有n只猫咪，开始时每只猫咪有花生米0颗，先有一组操作：
由下面三个中的k个操作组成：

*  g i 给i只猫咪一颗花生米
*  e i 让第i只猫咪吃掉它拥有的所有花生米
* s i j 将猫咪i与猫咪j的拥有的花生米交换

  现将上述操作做m次后，问每只猫咪有多少颗花生米？

 

#分析
因m的数据范围较大，用矩阵连乘。

构建矩阵模型，peanut[N] = {0,0，。。。。0,1}：即前n个数为0,最后一个数取1
matrix[N][N],初始化条件下为单位矩阵，。。。

对猫咪进行操作转化为在对矩阵peanut进行操作，一组操作过程转化为矩阵matrix,那么m次操作，即对peanut*(matrix^m)

```
EXP:
input:
316
g 1
g 2
g 2
s 12
g 3
e 2
```

初始化下矩阵:peanut  
0001 即每只猫咪的花生米个数为0

初始化下matrix为单位矩阵

```
1000
0100
0010
0001
```
经过操作 
```
g 1
```

给1号1颗花生米，即在第一列的最后一行加1

```
1000
0100
0010
1001

g 2
1000
0100
0010
1101

g 2
1000
0100
0010
1201

s 12
//即交换第1,2列
0100
1000
0010
2101

g 3
0100
1000
0010
2111

e 2
//将第2列全部置为0
0000
1000
0010
2011


```


最后peanut = peanut＊matrix＊matrix.....＊matrix = peanut＊(matrix^m)故可用矩阵快速求幂
peanut的前n个数即为每只猫咪拥有的花生米数

＃Answer

```c++
#include<stdio.h>
#include<string.h>
constint N =100+5;
struct Matrix
{
 __int64 matrix[N][N];
 __int64 row,coloumn;
 Matrix(){
 memset(matrix,0,sizeof(matrix));
 }
};

Matrix getE(__int64 n)//获取n*n的单位矩阵
{
 Matrix matrix;
 matrix.coloumn=n;
 matrix.row = n;
 int i;
 for(i=0;i<n;i++)
 {
  matrix.matrix[i][i]=1;
 }
 return matrix;
}

Matrix initP(__int64 n)//初始化花生米矩阵
{
 Matrix matrix;
 matrix.coloumn = n;
 matrix.row =1;
 matrix.matrix[0][n-1]=1;
 return matrix;
}

Matrix mutiply(Matrix a,Matrix b)//返回a*b
{
 __int64 i,j,k;
 Matrix ans;
 ans.row = a.row;
 ans.coloumn = b.coloumn;
 for(i=0;i<a.row;i++)
   for(k=0;k<a.coloumn;k++)
    if(a.matrix[i][k])//优化
for(j=0;j<b.coloumn;j++)
      {
  
         ans.matrix[i][j]=ans.matrix[i][j]+a.matrix[i][k]*b.matrix[k][j];
      }

      return ans;
}

int main() 
{
 __int64 n,m,k;
 while(scanf("%I64d%I64d%I64d",&n,&m,&k)!=EOF)
 {
  Matrix matrix;//操作矩阵
        Matrix peanut;//花生米
if(n==0&&m==0&&k==0)break;
       matrix = getE(n+1);
    peanut = initP(n+1);
    
    __int64 i;
       char op[5];
    __int64 x,y;
    while(k--)
    {
  
   scanf("%s",op);
   if(op[0]=='g')
   {
             scanf("%I64d",&x);//给x一颗花生米
    matrix.matrix[n][x-1]++;
   }
   else
    if(op[0]=='e')
    {
                scanf("%I64d",&x);//x吃掉所有的花生米
for(i=0;i<=n;i++)
     matrix.matrix[i][x-1]=0;
    }
    else
    {
     scanf("%I64d%I64d",&x,&y);//交换x与y的花生米
for(i=0;i<=n;i++)
     {
      __int64 tt = matrix.matrix[i][x-1];
      matrix.matrix[i][x-1]=matrix.matrix[i][y-1];
      matrix.matrix[i][y-1] = tt;
     }
    }
   
    }
      
   
   
    while(m)          
    {
     if(m&1)
     {
     peanut = mutiply(peanut,matrix);
     }    
    m>>=1;
    matrix = mutiply(matrix,matrix);
    }

    printf("%I64d",peanut.matrix[0][0]);
    for(i=1;i<n;i++)
     printf(" %I64d",peanut.matrix[0][i]);
    printf("\n");

 }

 return0;
}
```