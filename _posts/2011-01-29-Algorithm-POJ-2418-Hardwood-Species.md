---
layout: post
title: "POJ 2418 Hardwood Species【二叉查找树】"
keywords: ["algorithm", "POJ"]
description: "POJ 2418 Hardwood Species 解题报告"
category: "algorithm"
tags: ["ACM","Search"]

---

[POJ 2418 Hardwood Species]()
#算法核心 
二叉查找树
#题目大意：
通过卫星得到了某一个区域的树名，将这些树名按字典顺序输出，
并输出在树的总数中所占的比例，保留小数点后四位。

#主要思想：
对每种树做统计，并计算出所占的比例并不难。
难的是如何在规定时间内按字典顺序输出输入中涉及的树名。
字典顺序可以启发我们用排序的方法解决，我们可以把树名
作为关键字来比较大小，而strcmp函数也给了我们比较大小提
供了条件。接下来就是要解决时间问题。如果用插入排序的算法
由于大量的数据需要大量的比较，就会超时。所以这里借助了比较
经典的数据结构，二叉查找树。那么我们就可以先对输入建树，然
后再通过树的中序遍历来输出结果。而比例的计算可以在树的节点
中增加一个空间，
用于存储关键字出现的次数。

#Answer
```
#include<stdio.h>
#include<string.h>
constint NameLen =40;
long total ;//树的总数
struct treeNode
{
 char name[NameLen];
 int count;//该树出现次数
    treeNode *left,*right;
};

void insert(treeNode *root,char*name)//将名字插入对应的位置
{
   treeNode *iNode = root;
   bool find =false;
   while(!find)
   {
      int cmp = strcmp(name,iNode->name);
   if(cmp ==0)//名字相同
   {
    (iNode->count)++;
    return;
   }

   if(cmp<0)//在左子树
   {
         if(iNode->left==NULL)
   {
   treeNode *h;
   h=new treeNode;
   h->count=1;
   strcpy(h->name,name);
   h->left = NULL;
   h->right = NULL;
      iNode->left = h;
   return;
   }
   iNode = iNode->left;
   }

   else
    if(cmp>0)
    {
     if(iNode->right == NULL)
     {
    treeNode *h;
    h=new treeNode;
    h->count=1;
    strcpy(h->name,name);
    h->left = NULL;
    h->right = NULL;
    iNode->right = h;
    return;
     }
     iNode = iNode->right;
    }
   }
}

void output(treeNode *root)
{
   if(root->left!=NULL)
    output(root->left);
   printf("%s %.4lf\n",root->name,root->count*100.0/total);
   if(root->right!=NULL)
    output(root->right);
}

int main()
{
 char name[NameLen];
 gets(name);
 treeNode *root;
 root =new treeNode;
 root->left = NULL;
 root->right = NULL;
 root->count =1;
 strcpy(root->name,name);
  total =1;
 while(gets(name)!=NULL)
 {
  insert(root,name);
  total++;
 }
     

 output(root);

return0;
}
```
