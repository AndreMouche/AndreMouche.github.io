---
layout: post
title: "TiDB 扩缩容原理及常见问题"
keywords: ["tidb"]
description: "TiDB 扩缩容原理及常见问题"
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

本文作为本系列文章第一篇，将以扩容为例子，介绍扩容过程中主要涉及的资源两部分内容：
- 调度产生原理
- 调度执行原理


# 扩容 TiKV 原理及常见问题

一般的，当集群中 TiKV 资源跑到 75% 左右时，一般的调优手段无法解决资源使用上的瓶颈，此时，我们就需要通过 添加 tikv 节点的方式，来提高集群的整体性能。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale-out-summary.png?raw=true" width="600" />

如图，当集群只有 三个 tikv 时，能够使用的存储、CPU 、 memory 到达使用瓶颈时，我们可以通过加节点的方式增加集群相关资源。
下面我们简单来看一下，三 tikv 节点下，增加一个 tikv 节点时，TiDB 集群是如何让新节点的物理资源能够被集群使用起来的。
首先在三个 TiKV 节点的集群中，在我们 PD 的统一调度下，会尽量将三个 tikv 节点的资源得到均衡使用。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale-out-summary1.png?raw=true" width="600" />

新加入一个节点时，PD 会慢慢的将相关数据以 Region 为单位迁移到新节点上，以充分使用新节点的 存储、memory 和 CPU 等资源。这当中最重要也是最耗时的就是将 Region 的副本数据从老节点搬迁到新节点上。如下图，PD 将 Region 1 的其中一个副本从 `store-3` 搬迁到了新节点 `store-4` 上，这个过程我们叫 `balance-region`.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale-out-summary2.png?raw=true" width="600" />

## `Balance region` 具体步骤

`Balance region` 以 region 为单位执行，由 PD 计算需要搬迁的 region, 由 TiKV 执行。 具体执行过程，简单的来说，主要分为以下三个步骤：
1. 新节点上新增副本 `learner` 节点（leaner 节点不参与投票过程）。
2. 将新节点的 `learner` 角色和老节点的 `follower` 角色互换
3. 将老节点上的副本（`learner`） 删除
以上步骤最终会变成一条条调度指令，下发给 KV 去执行，下面我们来看每个调度指令是如何从进行的：

### Step1: Add learner

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/add_learner.png?raw=true" width="600" />

1. PD 中 `balance-region-scheduler` 生成 `balance-region` operator, 制定具体执行步骤。
2. PD 通过心跳的方式，告诉 `leader` 节点执行 `add learner peer` 操作
3. Leader 所在节点在这个 Region 的 `raft-group` 里面广播这个消息，并最终从 `leader` 上生成 `snapshot` 发送给 `store-4` , 添加 `learner` 节点完成。
4. Leader 收到来自 `learner` 的消息，上报心跳给 PD 告知 add learner 这一步骤执行成功

### Step2: Switch role

`store-4` 上的 `learner` 节点虽然有完整的数据，但不参与投票过程。我们期望是将 `store-3` 上的 `follower` 迁移到 `store-4` 上，因此，在这一步骤，我们会将 `store-4` 和 `store-3` 上的副本的角色互换，最终 `store-3` 上的老副本会变成 `learner`, 而新节点上的副本角色变成 `voter` 也就是 `follower`.
这个过程为了保证数据的安全性，实际情况分为两条调度指令执行。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/switch_role.png?raw=true" width="600" />

具体执行步骤为：
1. PD 通过心跳知道 `learner` 添加成功后，发送角色互换的指导给 `leader` 节点
2. Leader 节点在 `raft-group` 里面广播消息
3. 所有副本收到该心跳后，更新自己的配置。至此，该 `raft-group` 完成角色互换的配置更新。

这个过程因为只涉及逻辑变更，很快，也很难出问题。

### Step3: Remove old peer

经过步骤 2，`store-4` 上已经有一个 `follower`,  且已经不会有数据访问到 `store-3` 上的 `learner`, 因此我们可以安全的删除这个副本。具体执行步骤如下：
1. PD 发送心跳给 `leader`, 告知删除 `store-3` 上的副本
2. Leader 在当前 `region` 的 `raft-grou`p 里面广播这个消息
3. 所有副本在收到这个消息后，更新自己的配置信息，而 `store-3` 则会将自己这个副本完全删除。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/remove_old_peer.png?raw=true" width="600" />

## PD 上关键日志

以上过程，我们可以从 PD 的日志中看到完整的执行情况：

### create `balance-region` operator

``` 
// create balance-region operator
[2024/07/09 16:25:23.105 +08:00] [INFO] [operator_controller.go:488] ["add operator"] [region-id=142725] [operator="\"balance-region {mv peer: store [1] to [166543141]} (kind:region, region:142725(575, 5), createAt:2024-07-09 16:25:23.105733052 +0800 CST m=+1209042.707881308, startAt:0001-01-01 00:00:00 +0000 UTC, currentStep:0, size:92, steps:[0:{add learner peer 166543143 on store 166543141}, 1:{use joint consensus, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner}, 2:{leave joint state, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner}, 3:{remove peer on store 1}], timeout:[17m0s])\""] [additional-info="{\"sourceScore\":\"817938.92\",\"targetScore\":\"4194.83\"}"]
```

###  Step1: `add learner peer` on new store

```
// Step1: add learner node 
[2024/07/09 16:25:23.105 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="add learner peer 166543143 on store 166543141"] [source=create]
[2024/07/09 16:25:23.107 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=142725] [detail="Add peer:{id:166543143 store_id:166543141 role:Learner }"] [old-confver=5] [new-confver=6]
[2024/07/09 16:25:23.108 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="add learner peer 166543143 on store 166543141"] [source=heartbeat]
```

### Step2/Step3: use joint consensus to switch role

```
// Step2: use joint consensus
[2024/07/09 16:25:24.248 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="use joint consensus, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner"] [source=heartbeat]
[2024/07/09 16:25:24.250 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=142725] [detail="Remove peer:{id:142726 store_id:1 },Remove peer:{id:166543143 store_id:166543141 role:Learner },Add peer:{id:142726 store_id:1 role:DemotingVoter },Add peer:{id:166543143 store_id:166543141 role:IncomingVoter }"] [old-confver=6] [new-confver=8]
// Step3: leave joint state
[2024/07/09 16:25:24.250 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="leave joint state, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner"] [source=heartbeat]
[2024/07/09 16:25:24.252 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=142725] [detail="Remove peer:{id:142726 store_id:1 role:DemotingVoter },Remove peer:{id:166543143 store_id:166543141 role:IncomingVoter },Add peer:{id:142726 store_id:1 role:Learner },Add peer:{id:166543143 store_id:166543141 }"] [old-confver=8] [new-confver=10]
```

### Step4: remove old peer

```
// Step4: remove learner node
[2024/07/09 16:25:24.252 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="remove peer on store 1"] [source=heartbeat]
[2024/07/09 16:25:24.255 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=142725] [detail="Remove peer:{id:142726 store_id:1 role:Learner }"] [old-confver=10] [new-confver=11]

```

### PD report operator is finished

```
// operator finished
[2024/07/09 16:25:24.255 +08:00] [INFO] [operator_controller.go:635] ["operator finish"] [region-id=142725] [takes=1.149978179s] [operator="\"balance-region {mv peer: store [1] to [166543141]} (kind:region, region:142725(575, 5), createAt:2024-07-09 16:25:23.105733052 +0800 CST m=+1209042.707881308, startAt:2024-07-09 16:25:23.10595495 +0800 CST m=+1209042.708103209, currentStep:4, size:92, steps:[0:{add learner peer 166543143 on store 166543141}, 1:{use joint consensus, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner}, 2:{leave joint state, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner}, 3:{remove peer on store 1}], timeout:[17m0s]) finished\""] [additional-info="{\"cancel-reason\":\"\",\"sourceScore\":\"817938.92\",\"targetScore\":\"4194.83\"}"]
``` 

在具体的运维过程中，当发现上线速度缓慢时，如果有大量的 `balance-region` operator 执行超时，我们可以通过查看 PD 日志中具体某个 operator 的执行情况，对照着以上日志，来看 `balance-region` 具体卡在哪一步。

## 实时监控扩容速度

从上文我们知道，扩容 TiKV 的速度主要取决于数据的搬迁情况，主要分为两个因素：
1. PD 产生 `balance-region`  operator 的速度
2. TiKV 消费 `balance-region` 的速度，也就是物理数据的搬迁速度。

在扩容后，要做到整个集群的资源的快速均匀，一般跟我们 region 的数量、size 有直接关系，当然与我们集群的繁忙程度也有关系。目前，PD 提供了一些监控可以让我们看到扩容的状态，主要有

- PD->metrics:
  - Region health
    - Pending-region-count: 正准备 `add learner` 但还没添加成功的副本
    - Learner-peer-count: 当前集群中 `learner` 副本的个数，需要注意的是，如果集群本身有 `Tiflash` 节点，这个数量也包含了 `Tiflash` 节点里的 `learner` 个数。
  - Speed：
    - Online store progress：当前扩容进度，根据剩余需要 balance 的空间计算得出
    - Left time: 预估剩余时间，根据剩余需要 balance 的空间及当前数据搬迁速度得出
    - Current scaling speed: 当前扩容数据搬迁实际速度

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale_speed.png?raw=true" width="600" />


## Summary

从上文介绍我们知道，对于扩容来说，主要需要考虑的是新加节点进来后，集群数据通过调度，重新让所有在线的 tikv 的资源使用到达一个平衡的状态。对于缩容来说，原理也是类似的。
影响扩容的主要原因无非就是以下两点：
- PD 侧感知集群资源变化并生成相应调度计划（operator）的速度
- 调度计划执行的速度，其中最主要的还是 TiKV 侧副本数据的搬迁

因此接下来，我们将重点介绍扩缩容过程中的核心模块及常见问题，分为以下几个部分：
- [扩容过程中调度生成原理及常见问题](https://andremouche.github.io/tidb/tidb-scale-out-pd.html)
- [缩容过程中调度生成原理及常见问题](https://andremouche.github.io/tidb/tidb-scale-in.html)
- [扩缩容过程调度执行（TiKV 副本搬迁）的原理及常见问题](https://andremouche.github.io/tidb/tidb-move-region-between-stores.html)
