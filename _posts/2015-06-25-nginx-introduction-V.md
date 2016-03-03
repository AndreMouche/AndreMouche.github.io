---
layout: post
title: "Nginx介绍(译文V--总结)"
keywords: ["nginx"]
description: "nginx"
category: "nginx"
tags: ["nginx"]
author: Shirly

---


## Nginx 介绍（译文V--总结）


* 原文 [nginx](http://www.aosabook.org/en/nginx.html)
* 作者 [Andrew Alexeev](http://www.aosabook.org/en/intro2.html#alexeev-andrew)

<ul>
译文结构
<ul>
<li><a href="/nginx/nginx-introduction-I.html">译文I--为什么高并发很重要</a></li>
<li><a href="/nginx/nginx-introduction-II.html">译文II--Nginx架构综述</a></li>
<li><a href="/nginx/nginx-introduction-III.html">译文III--Nginx配置结构</a></li>
<li><a href="/nginx/nginx-introduction-IV.html">译文IV--深入nginx内核</a></li>
<li><a href="/nginx/nginx-introduction-V.html">译文V--总结</a></li>
</ul>
</li>
</ul>


## 总结

Igor Sysoev开始编写nginx时，大部分构建互联网的软件都已经存在，且这些软件的架构通常遵循以下几点定义：

* 传统服务器
* 网络硬件
* 操作系统
* 通用的传统互联网架构。

然而这并未阻止Igor考虑去提升web服务器领域相关工作。所以，第一个教训显而易见的就是：总是存在可提升空间。

带着开发更好web软件的想法，Igor花了不少时间设计开发最初的代码结构，并研究在多个操作系统下尝试不同的方法去优化代码。在1.0版本历时十年的活跃开发后，Igor开发了2.0版本原型。显然一个新架构的初始原型和代码结构，对于软件产品的后续开发极其重要。

另外值得提到的一点是要保持专注。Nginx的windows版本是个好例子，说明无论在开发者的核心技能或应用目标上避免过于分散开发工作是值得的。同样地，加强nginx重写引擎对现存遗留配置的后向兼容能力，这些努力也是值得的。

最后值得提到的是，尽管nginx开发者社区并不大，nginx的第三方模块和扩展一直都是nginx受欢迎的重要因素。Nginx用户社区和作者们十分感谢Evan Miller, Piotr Sikora, Valery Kholodkov, Zhang Yichun (agentzh)以及其他优秀软件工程师所做的工作。


This work is made available under the [Creative Commons Attribution 3.0 Unported](http://creativecommons.org/licenses/by/3.0/legalcode) license. Please see the [full description of the license](http://www.aosabook.org/en/intro1.html#license) for details
