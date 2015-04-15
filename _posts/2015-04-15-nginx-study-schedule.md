---
layout: post
title: "Nginx学习计划"
keywords: ["nginx", "schedule"]
description: "nginx study schedule"
category: "nginx"
tags: ["nginx"]

---


##Nginx学习计划

<ul>
<li><a href="#nginx学习计划">Nginx学习计划</a><ul>
<li><a href="#概要">概要</a></li>
<li><a href="#hello-nginx(第一阶段)">Hello Nginx(第一阶段)</a></li>
<li><a href="#编写nginx模块(第二阶段)">编写Nginx模块(第二阶段)</a></li>
<li><a href="#深入nginx上(第三阶段)">深入Nginx上（第三阶段）</a></li>
<li><a href="#深入nginx下(第四阶段)">深入Nginx下(第四阶段)</a></li>
</ul>
</li>
</ul>



##概要

本文将介绍为期两个月的nginx学习计划，其中nginx基于版本[nginx-1.6.3.tar.gz](http://nginx.org/download/nginx-1.6.3.tar.gz)展开学习，主要资料除源码外，还包括《深入理解nginx模块开发与设计》,Nginx官方doc [Introduction](http://nginx.org/en/docs/ )

##Hello Nginx(第一阶段)

###学习目标

熟悉nginx的应用场景，熟练掌握配置

###学习资料

*  Nginx官方doc [Introduction](http://nginx.org/en/docs/ )
* [nginx的几个重要的应用场景](http://www.aosabook.org/en/nginx.html)
* 深入理解nginx模块开发与架构设计 第1-2章

###学习大纲

学习要点如下：

 *  HTTP Server --- 静态站点Http服务器, 图片, 文本,等所有HTTP资源.
 *  Http Proxy --- Http代理服务器。  区分代理和反向代理。 cgi代理, wsgi代理， fastcgi代理
 *  七层负载均衡。 某个节点挂掉了之后，可以将请求自动转到另一个节点
 *  URL路径资源rewrite 
 *  caching 
 *  bandwidth control

##编写Nginx模块(第二阶段)

###学习目标

 熟悉Nginx HTTP模块相关数据结构，编写一个简单的HTTP过滤模块

###学习资料

* 深入理解Nginx模块开发与设计 第3-7章
* [nginx-1.6.3.tar.gz](http://nginx.org/download/nginx-1.6.3.tar.gz) 源码目录:nginx-1.6.3/src/core

###学习大纲

* 开发一个简单的HTTP模块
* 配置、error日志和请求上下文
* 访问第三方服务
* 开发一个简单的HTTP过滤模块
* Nginx提供的高级数据结构

##深入Nginx上(第三阶段)

###学习目标

熟悉Nginx 基础架构、事件模块、HTTP框架

###学习资料

* 深入理解Nginx模块开发与设计 第8-11章
* 源码目录：nginx-1.6.3/src/event、nginx-1.6.3/src/http

###学习大纲

* Nginx基础架构
* 事件模块
* HTTP框架的初始化
* HTTP跨家的执行流程

##深入Nginx下(第四阶段)

###学习目标

了解upstream机制的设计与实现、邮件代理模块、进程间的通讯机制

###学习资料

* 深入理解Nginx模块开发与设计 第12-14章
* 源码目录：nginx-1.6.3/src/mail、nginx-1.6.3/src/http、nginx-1.6.3/src/event

###学习大纲

* upstream机制的设计与实现
* 邮件代理模块
* 进程间的通讯机制


