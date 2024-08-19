---
layout: post
title: "TiDB 扩容过程中 PD 生成调度的原理和常见问题"
keywords: ["tidb"]
description: "TiDB 扩容过程中 PD 生成调度的原理和常见问题"
category: "tidb"
tags: ["tidb","tikv","pd"]
comments: true
---


作为一个分布式数据库，扩缩容是 TiDB 集群最常见的运维操作之一。本系列文章，我们将基于 v7.5.0 具体介绍扩缩容操作的具体原理、相关配置及常见问题。
扩缩容从集群视角考虑，主要需要考虑的是扩缩容完成后，集群数据通过调度，让所有在线 tikv 的资源使用到达一个均衡的状态。在这个过程中，主要涉及以下两个关键步骤：
- 调度产生速度
- 调度执行速度

本系列文章我们将围绕以上两个逻辑，重点介绍扩缩容过程中的核心模块及常见问题，分为以下几个部分：
- [TiDB 扩容原理及常见问题](https://andremouche.github.io/tidb/tidb-scale-in-and-out.html)
- [扩容过程中调度生成原理及常见问题](https://andremouche.github.io/tidb/tidb-scale-out-pd.html)
- [缩容过程中调度生成原理及常见问题](https://andremouche.github.io/tidb/tidb-scale-in.html)
- [扩缩容过程调度执行（TiKV 副本搬迁）的原理及常见问题](https://andremouche.github.io/tidb/tidb-move-region-between-stores.html)

本文我们重点介绍 TiDB 扩容过程中 PD 生成调度的原理和常见问题，对于扩容过程中调度消费的原理及常见问题，可以看[这篇](https://andremouche.github.io/tidb/tidb-move-region-between-stores.html)文章。


# 扩容过程中 PD 生成调度的原理及常见问题

因为本身 PD 就有一系列监控和平衡各个 TiKV 之间的资源使用情况的调度器，因此 PD 没有针对扩容给出单独的调度器。目前这类调度器主要有两类：
- `Balance-region-scheduler`：负责将 Region 均匀的分散在集群中所有的 store 上，主要用于分散存储压力
- `Balance-leader-scheduler`: 负责将 region leader 均匀分布在 store 上，主要负责分散客户端的请求压力（CPU）

因此我们也可以认为，在扩容过程中，负责生成调度的主要是以上两个调度器在发挥作用。
对于 `balance-leader-scheduler`, 因为没有数据搬迁，只是 `raft-group` 元数据的变更，因此特别快。一般情况下，我们不需要特别关注这个（也很少出问题）
本节将重点介绍 `balance-region-scheduler`, 也就是扩容情况下，迅速往新扩容 kv 上搬迁副本的调度器行为及常见问题。

首先我们来看一下 `balance-region-scheduler` 是如何选择并生成 `balance-region` `operator` 的：

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/balance_region_schedule.png?raw=true" width="600" />

## 调度原理

Balance-region-scheduler 每隔 10ms~5s/10s 会发起一次调度。 `balance-region-scheduler` 每次生成调度的具体逻辑如下：

1. 检查 [region-schedule-limit](https://docs.pingcap.com/tidb/v7.5/pd-configuration-file#region-schedule-limit), 用于控制当前 balance-region operator 的并发数量。
2. 选择本次要搬走的 store
   - 过滤选出可以作为数据源搬迁的 store, 过滤条件有：
        - 检查 store-limit 是否符合条件：[store limit remove-peer](https://docs.pingcap.com/tidb/stable/configure-store-limit#principles-of-store-limit-v2) 
        - 检查这个 store 上的 [max-snapshot-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-snapshot-count) 是否超过
   -  在符合要求的 store 里面，选出最适合搬走副本的 store, 按照 region_score 倒叙排序, 优先考虑的条件有：
        - 空间不足时，优先选剩余空间最不足的节点 （使得 tikv 的剩余数据量均衡）
        - 空间富裕时，选择已用空间最多的节点（使得 tikv 的数据量分布均衡）
        - 中间状态综合考虑两个因素
3. 选择要搬走的副本，从当前选中的 source store 中选择一个副本，选择条件按优先级如下：
    - pending regions
    - followers
    - Leaders
    - Learners
4. 选择一个目标 store 作为当前副本的目标
    - 通过 placement-rule 选择符合要求的 store
    - 检查 [store limit add-peer](https://docs.pingcap.com/tidb/stable/configure-store-limit#principles-of-store-limit-v2)
    - 检查 [max-snapshot-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-snapshot-count) 
    - 检查 [max-pending-peer-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-pending-peer-count)
    - 最后对以上条件都符合的目标 store , 选择 region_score 最小的那个节点

## 常见问题

接下来，我们将根据上文说的调度生成的顺序，来介绍过程中可能遇到的问题及解决方案。

### scheduler 被关闭导致没有调度生成

默认情况下 `balance-region-scheduler` 会被启用，但确实能够用 `pd-ctl` 将其删除。当发现没有 `balance-region` operator 生成时，第一步需要检查的便是确认 `balance-region-scheduler` 是否被启用。

- 通过监控查看 balance-region-scheduler 是否被启用，即调度发生的频率：
    - PD->scheduler->scheduler is running
- 通过 pd-ctl 查看当前正在运行的 schedule

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/check_scheduler.png?raw=true" width="600" />

- 编辑 balance-region-scheduler
``` 
// remove balance-region-scheduler
~$ tiup ctl:v7.5.2 pd schedule remove balance-region-scheduler
Starting component ctl: /home/tidb/.tiup/components/ctl/v7.5.2/ctl pd schedule remove balance-region-scheduler Success!
// add balance-region-scheduler
~$ tiup ctl:v7.5.2 pd schedule add balance-region-scheduler
Starting component ctl: /home/tidb/.tiup/components/ctl/v7.5.2/ctl pd schedule add balance-region-scheduler Success! The scheduler is created."
```

### 受限于 [region-schedule-limit](https://docs.pingcap.com/tidb/v7.5/pd-configuration-file#region-schedule-limit) 

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/region_schedule_limit.png?raw=true" width="600" />

`region-schedule-limit` 是用来控制 balance-region / scatter-range/hot-region 等 region 相关的 operator 生成的并发度的，默认值是 2048，一般很难到达瓶颈。但考虑到这个参数是可以修改的，有时候误操作会将这个参数改得比较小，就容易出现问题。

- 通过 pd-ctl 检查及设置

``` 
// check config
tidb@~:~$ tiup ctl:v7.5.2 pd config show
Starting component ctl: /home/tidb/.tiup/components/ctl/v7.5.2/ctl pd config show
{
  .....
  "schedule": {
    .... 
    "max-movable-hot-peer-size": 512,
    "max-pending-peer-count": 64,
    "max-snapshot-count": 64,
    "max-store-down-time": "30m0s",
    "max-store-preparing-time": "48h0m0s",
    "merge-schedule-limit": 0,
    "patrol-region-interval": "10ms",
    "region-schedule-limit": 2048,
    .... 
  }
}

// update region-schedule-limit
tidb@~:~$ tiup ctl:v7.5.2 pd config set region-schedule-limit 1024
Starting component ctl: /home/tidb/.tiup/components/ctl/v7.5.2/ctl pd config set region-schedule-limit 1024
Success!
``` 

- 通过监控看是否遇到瓶颈: PD ->operator->schedule reach limit

### 检查要被 move peer 的 store 

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/source_store.png?raw=true" width="600" />

一般在 scale out 的过程中，已用空间比较大的 store 很容易被选为 balance-region 的目标对象，因此已用空间比较大的那个 store 很容易受到 store limit remove-peer 的影响。

- 通过监控查看 store limit 配置：PD->cluster->PD scheduler config/store limit

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/config.png?raw=true" width="600" />

- Store limit remove peer 不足的场景：PD->scheduler->filter source 看到大量的 balance-region-XXX-remove-limit 时：

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/store-limit-remove-peer.png?raw=true" width="600" />

### 检查待 add peer 的 store

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/max-snapshot.png?raw=true" width="600" />

在扩容场景下，这类 store 往往是新扩容的节点，因此这些新节点很容易变成热点，相关配置也比较容易到达使用瓶颈。
这里相关配置主要是以下三个：
  - [store limit add-peer](https://docs.pingcap.com/tidb/stable/configure-store-limit#principles-of-store-limit-v2) : speed limit for the special store
  - [max-snapshot-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-snapshot-count) : when the number of snapshots that a single store receives or sends meet the limit, it will never be chosen as a source or target store
  - [max-pending-peer-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-pending-peer-count): control the maximum number of pending peers in a single store.  


- Metrics 中查看配置项(同 source store 配置项查看)：pd->cluster->pd scheduler config/store limit
- 查看目标节点配置项是否遇到瓶颈：pd->schedule -> filter target -> balance-regionXXX-{config}-filter

### 生成 operator 的速度

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/create_operator_speed.png?raw=true" width="600" />

经过以上步骤，一个 balance region 的 operator 变生成了。我们可以通过 pd->operator->schedule operator create -> balance region 查看当前 operator 的产生速度.

## Summary 

PD 生成 balance region operator 的速度，直接影响整个扩容的速度。为了保证 PD 在生成 operator 的速度不会成为瓶颈，我们可以根据上文中的监控来确定以下配置项是否设置合理，进行适当的调优：
- ensure `balance-region-scheduler` is running
- [region-schedule-limit](https://docs.pingcap.com/tidb/v7.5/pd-configuration-file#region-schedule-limit) ：control the generation speed of schedule-region operator 
- [store limit remove-peer/add-peer](https://docs.pingcap.com/tidb/stable/configure-store-limit#principles-of-store-limit-v2) : speed limit for the special store
- [max-snapshot-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-snapshot-count) : when the number of snapshots that a single store receives or sends meet the limit, it will never be choosed as a source or target store
- [max-pending-peer-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-pending-peer-count): control the maximum number of pending peers in a single store.  

### 如何判断当前扩容的瓶颈在 TiKV 还是在 PD上

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale_speed_limit.png?raw=true" width="600" />

我们可以通过对比 PD 上的以上两个监控：

- PD->operators->schedule operator create
- PD->operators->schedule operator finish 

来判断 operator 的消费速度能否跟得上 生成速度，如果不能，说明 TiKV 中出现了瓶颈，则需要继续从 TiKV 中去寻找答案。
