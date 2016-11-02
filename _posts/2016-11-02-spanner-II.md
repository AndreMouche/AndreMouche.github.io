---
layout: post
title: "[译文 II]Spanner:Google's Globally-Distributed Database"
keywords: ["Spanner"]
description: "[译文 II]Spanner:Google's Globally-Distributed Database"
category: "spanner"
tags: ["Spanner","Google"]
comments: true
---
# [译文II]Spanner:Google's Globally-Distributed Database

## 原文
[Spanner:Google's Globally-Distributed Database](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/spanner-osdi2012.pdf)

## 译文结构

* [摘要、介绍](http://andremouche.github.io/spanner/spanner-I.html)
* [实现一](#)
* TODO

## 实现

本章将介绍以下内容：

* 介绍Spanner的架构及其实现的基本原理,
* 然后，将介绍用于管理副本和位置的目录抽象，以及移动数据的单位。
* 最后，将介绍我们的数据模型，为何Spanner看起来像关系型数据库而不是一个键值存储系统（key-value store）,以及用户如何控制数据的位置。

一个Spanner的部署被称为一个universe。考虑到Spnner在全球范围内管理数据，其只有少量正在运行的universes。目前我们有个用于测试／实验(test/playground)的unverise，一个开发／产品(developing/production) unverise,和一个只服务生产环境的universe

Spanner由一个zone的集合组成，每个zone是一个Bigtable服务集群部署的粗糙模拟。zone是管理部署的单位。zone的集合也是可以复制数据位置的集合。随着新的数据中心投入服务，zone被添加到正在运行的服务中；当旧数据中心被关闭时，zone被从正在运行的服务中移除。zone也是物理隔离的单位：一个数据中心可以包含一个或多个zone；例如，多个应用在同一个数据中时，必须被分布到不同的服务集群（zone）中。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/spanner_fig_1.jpg?raw=true" alt="spanner_fig_1" title="ngx_module_t.jpg" width="600" />




[Figure 1]展示了一个Spanner universe的服务器，简要介绍如下：

* 一个zone由一个zonemaster和100-上千台spanserver。zonemaster把数据分配给spanservers,spanserver为客户端提供数据。
* location proxy用来帮助客户端定位为其提供数据的spanserver。
* universe master和placement driver目前都只有一个
* universe master主要为一个管理zone的控制台，它显示所有zone的状态信息，用于互动调试。
* placement driver 用于处理zone之间分钟级别的数据自动移动。
* placement driver 周期性地与spanserver交互，为满足被更新的副本限制或负载均衡，发现需要移动的数据

由于篇幅限制，我们将只介绍spanserver的相关细节。