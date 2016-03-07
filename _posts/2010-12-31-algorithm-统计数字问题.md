---
layout: post
title: "统计数字问题"
keywords: ["algorithm", "算法设计与分析"]
description: "统计数字问题解题报告"
category: "algorithm"
tags: ["ACM"]
comments: true
---

# 统计数字问题

## Description

一本书的页码从自然数1 开始顺序编码直到自然数n。书的页码按照通常的习惯编排，
每个页码都不含多余的前导数字0。例如，第6 页用数字6 表示，而不是06 或006 等。数
字计数问题要求对给定书的总页码n，计算出书的全部页码中分别用到多少次数字0，1，
2，…，9。

给定表示书的总页码的10 进制整数n (1≤n≤10^9) 。计算书的全部页码中分别用到多少
次数字0，1，2，…，9。

每个文件只有1 行，给出表示书的总页码的整数n。

输出文件共有10行，在第k行输出页码中用到数
字k-1 的次数，k=1，2，…，10。

## Sample Input

```
11
```

## Sample Output

```
1
4
1
1
1
1
1
1
1
1
```

### Answer

```
#include<stdio.h>  
#include<string.h>  
constint N =10;  
int ans[N];  //ans[i]存放数字i出现的次数
char str[N];  //输入的数字
int a[N],len; //a[i]为10^i，len为数字的长度 

void solve(long n)  //统计数字n
{  
	long m=n+1;  
	memset(ans,0,sizeof(ans));  
	int j,i;  
	for(i=0;i<len;i++)  
	{  
		int x=str[i]-'0';  
		int t=(m-1)/a[len-i-1];  
		ans[x]+=m-t*a[len-i-1];//自左往右到i位数字不变条件下，i位为x的数字个数  
		t=t/10;  
		j=0;  
		while(j<x)  
		{  
			ans[j]+=(t+1)*a[len-i-1];//统计当前位置为j出现的个数  
			j++;  
		}  
		while(j<N) //统计当前位置为j的数目 
		{  
			ans[j]+=t*a[len-i-1];  
			j++;  
		}  
		ans[0]-=a[len-i-1];//消去前导0  
	}  
	for(i=0;i<N;i++)  
		printf("%d\n",ans[i]);  
}  

int main()  
{  
	int i;  
	a[0]=1;  
	for(i=1;i<N;i++)  
		a[i]=a[i-1]*10;  
	long n;  
	while(scanf("%s",str)!=EOF) {  
		n=0;  
		len=strlen(str);  
		for(int i=0;i<len;i++)  
			n=n*10+str[i]-'0';  
		solve(n);  
	}  
	return0;  
}

```
