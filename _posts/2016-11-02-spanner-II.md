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

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/spanner/spanner1.jpg?raw=true" alt="spanner_fig_1" title="ngx_module_t.jpg" width="600" />




[Figure 1]展示了一个Spanner universe的服务器，简要介绍如下：

* 一个zone由一个zonemaster和100-上千台spanserver。zonemaster把数据分配给spanservers,spanserver为客户端提供数据。
* location proxy用来帮助客户端定位为其提供数据的spanserver。
* universe master和placement driver目前都只有一个
* universe master主要为一个管理zone的控制台，它显示所有zone的状态信息，用于互动调试。
* placement driver 用于处理zone之间分钟级别的数据自动移动。
* placement driver 周期性地与spanserver交互，为满足被更新的副本限制或负载均衡，发现需要移动的数据

由于篇幅限制，我们将只介绍spanserver的相关细节。

### 2.1 Spanserver软件栈

本节将专注于spanserver的实现，描述基于Bigtable,Spanner如何实现复制和分布式事务。软件栈信息显示如[Figure 2]。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/spanner/spanner2.jpg?raw=true" alt="spanner_fig_1" title="ngx_module_t.jpg" width="600" />



####  Tablet

在底部，每个spannserver负责管理100-1000称为tablet的数据结构实例:

* 一个tablet类似于Bigtable的tablet抽象，在spannerserver中，它实现了一系列如下映射关系：

	```
	(key:string,timestamp:int64)=> string 
	```

	不同于Bigtable的是，Spanner将时间戳关联到了数据－－这是一个关键的实现方式，它让Spanner比起一个key-vaue的存储，更像一个多版本数据库。

* 一个tablet的状态被存放于类B-tree文件集和一个write-ahead日志中，所有的这些又被存放于一个叫做Colossus（Google文件系统的继承者[15]）的文件系统里。


#### Paxos

为支持复制，每个spanserver在每个tablet之上实现了一个Paxos状态机，每个状态机将其状态及日志存放在其对应的tablet中。(早期的一个Spanner实现为支持更灵活的副本配置，支持一个table对应多个Paxos状态机。其复杂性迫使我们放弃了它)。

* 我们的Paxos实现支持常驻leaders(long-lived leaders),其基于时间leader租赁，长度一般在10秒。
* 当前的Spanner实现中，对每个Paxos写操作记录两次日志：一次在tablet的日志中，一次在Paxos的日志中。该做法乃权宜之计，我们会在将来对其改进。
* 我们的Paxos实现采用了管道式，因而提高了在广域网延时下的吞吐量，但paxos依旧会把写操作按顺序执行（事实上，我们将依赖于第四节－－Concurrency Control）。

Paxos状态机用来对一系列复制映射的持久化。

* 每个replica的key-value映射状态存放于其对应的tablet中
* 写操作必需在replica leader上初始化paxos协议
* 读操作可以直接从任意replica底层的tablet中访问状态信息，只要其足够新。
* replica的集合被统一称为一个**Paxos group**--paxos集合

#### Lock TABLE

在每个作为leader的replica上，spanserver设计了一个lock table以实现并发控制。该锁表包含了两阶段锁的状态。（注意，存在常驻Paxos leader是有效管理lock table的关键）。在Bigtable和Spanner中，我们都设计了长事务（long-lived transaction）（例如，报表的生成，耗时可能达几分钟。），当冲突存在时，乐观并发控制表现出很差的性能。对于需要同步的操作，如读事务，需要获得lock table里面的lock;其它操作便可忽略lock table。

#### Transaction manager

在每个作为leader的replica上，每个spanserver也实现了一个事务管理员（transaction manager）来支持分布式事务。

* 事务管理员用来实现一个participant leader（参与者领导），组里面的其它replica则被称为participant slave(参与者随从)。
* 如果一个事务只包含了一个paxos组（大部分事务都是如此），由于lock table和paxos一起已经提供了事务性，所以可以忽略transaction manager。
* 如果一个事务包含了多个paxos组，这些组的leader将协调执行两阶段提交。其中一个参与群会被选为coordinator leader:它的participant leader 被引用为coordinator leader,它的slaves则被称为coordinator slaves. 每个transaction manager的状态被存放于底层paxos组中（故被副本化了）