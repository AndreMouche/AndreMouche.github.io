---
layout: post
title: "Nginx介绍(译文III--Nginx配置结构)"
keywords: ["nginx"]
description: "nginx"
category: "nginx"
tags: ["nginx"]

---


##Nginx 介绍（译文III--Nginx配置结构）

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

##Nginx配置结构
Nginx配置系统的设计灵感来自于Igor Sysoev的Apache使用经验。他的主要观点是认为一个可扩展的配置管理系统是web服务器最基本要素。对于一个由大量虚拟服务器（virtual servers）、目录（durectirues）、位置（location）、数据集（datasets）组成的大型复杂配置而言，维护过程中很容易遇到扩展性问题。在一个相对较大的web设置、启动、管理工作，如果没有进行适当的设置，对于应用层和系统工程师而言，这无疑将是一个噩梦。

所以，nginx的配置结构按照如下两点为目标进行设计：

* 活在当下：简化日常维护
* 着眼未来：为web服务器的配置扩展提供一个简单易用的方法。

详细配置内容存放于一系列的存文本文件中，这些配置文件通常存放于/usr/local/etc/nginx和/etc/nginx下。主配置文件通常命名为nginx.conf。为了保持整洁，部分配置可放于单独文件中，并自动地被主配置文件引用。然而，需要注意的是，nginx目前并不支持Apache风格的分布式配置（如.htaccess等文件）。此外，所有和nginx web服务器行为相关的配置文件应统一集中指定。

配置文件由主(Master)进程在启动时读取和校验。由于worker进程是从master进程派生（fork）的，故woker能够获取到已编译好的只读配置，其中自动共享通过常用虚拟内存管理机制实现。

Nginx配置有多个不同的上下文，如：

* main
* http 
* server
* upstream
* location 
* (以及用于邮件代理的 mail )等指令块。

**上下文不会出现重叠**。例如，一个location指令块是不允许被放入main指令块中的。此外，为了避免引起不必要的歧义，不存在任何类似于“全局web服务器”的配置。Nginx配置诣在整洁和富有逻辑性，因而允许用户很容易去维护包含上千个指令的复杂的配置文件集。在一次私人会话中，Sysoev说：“全局服务器配置中的位置（location）、目录(directories)和其他模块（blocks）是Apache中我所不喜欢的特性，这就是不在nginx实现这些的原因。”

配置语法、格式及定义依照所谓的C风格协定，这种创建配置文件的方法已被广泛应用于大量开源及商业软件程序中。通过设计，C风格配置有以下特性：

* 适合嵌套描述
* 赋有逻辑性
* 易于创建、读取和维护
* 易于自动化
* 深受广大工程师喜爱

虽然nginx部分配置准则类似与Apahce的某些配置项，但是配置nginx实例与apache是完全不同的体验。例如，nginx支持重写规则，但同样的功能，Apache系统管理员要手动去适配其重写配置。同样地，重写引擎的实现也不相同。

此外，nginx的设置也提供了多种对底层机制的支持，这些往往是一个高效的web服务器配置中十分有用的模块。这里有必要简单提及nginx所有特有的变量和try_files指令：

* 为更好的控制来控制运行时的web服务器配置，Nginx开发了变量用于提供附加的增强（even-more-powerful）机制。变量为快速赋值做了优化，并且为内部预编译的索引。赋值是按需执行的，例如，变量的值在一个特定请求的生命周期中通常只计算一次，而后缓存起来。变量可被不同的配置指令使用，为描述条件请求处理行为提供了更多灵活性。
* try_files指令最初是为了在其更适应的场景下逐渐替换if条件配置语句，它的设计主要用于快速高效的尝试URI与内容之间的映射。总的来说，try_files指令很好用，并且极其高效和有用。强烈推荐读者完整地学习该指令，并在任何合适的地方使用它。



