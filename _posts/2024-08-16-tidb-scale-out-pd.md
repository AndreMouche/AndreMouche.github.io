---
layout: post
title: "TiDB 扩缩容原理及常见问题"
keywords: ["tidb"]
description: "TiDB 扩缩容原理及常见问题"
category: "tidb"
tags: ["tidb","tikv","pd"]
comments: true
---

作为一个分布式数据库，扩缩容是 TiDB 集群最常见的运维操作之一。本文，我们将基于 v7.5.0 具体介绍扩缩容操作的具体原理、相关配置及常见问题的排查。
# 扩容 TiKV 原理概要

一般的，当集群中 TiKV 资源跑到 75% 左右时，一般的调优手段无法解决资源使用上的瓶颈，此时，我们就需要通过 添加 tikv 节点的方式，来提高集群的整体性能。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale-out-summary.png?raw=true" width="600" />

如图，当集群只有 三个 tikv 时，能够使用的存储、CPU 、 memory 到达使用瓶颈时，我们可以通过加节点的方式增加集群相关资源。
下面我们简单来看一下，三 tikv 节点下，增加一个 tikv 节点时，TiDB 集群是如何让新节点的物理资源能够被集群使用起来的。
首先在三个 TiKV 节点的集群中，在我们 PD 的统一调度下，会尽量将三个 tikv 节点的资源得到均衡使用。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale-out-summary1.png?raw=true" width="600" />

新加入一个节点时，PD 会慢慢的将相关数据以 region 为单位迁移到新节点上，以充分使用新节点的 存储、memory 和 CPU 等资源。这当中最重要也是最耗时的就是将 region 的副本数据从老节点搬迁到新节点上。如下图，PD 将 region 1 的其中一个副本从 store-3 搬迁到了新节点 store-4 上，这个过程我们叫 balance-region.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale-out-summary2.png?raw=true" width="600" />

## Balance region 具体步骤

Balance region 以 region 为单位执行，由 PD 计算需要搬迁的 region, 由 TiKV 执行。简单的来说，主要分为以下三个步骤：
1. 新节点上新增副本 learner 节点。leaner 节点不参与投票过程。
2. 将新节点的 learner 角色和老节点的 follower 角色互换
3. 将老节点上的副本（learner） 删除
以上步骤最终会变成一条条调度指令，下发给 KV 去执行，下面我们来看每个调度指令是如何从进行的：

### Step1: Add learner
[Image]
1. PD 中 balance-region-scheduler 生成 operator balance region operator, 制定具体执行步骤。
2. PD 通过心跳的方式，告诉 leader 节点执行 add learner 操作
3. Leader 所在节点在这个 region 的 raft-group 里面广播这个消息，并最终从 leader 上生成 snapshot 发送给 store-4 , 添加 learner 节点完成。
4. Leader 收到来自 learner 的消息，上报心跳给 PD 告知 add learner 这一步骤执行成功

Step2: Swith role
store-4 上的 learner 节点虽然有完整的数据，但不参与投票过程。我们期望是将 store-3 上的 follower 迁移到 store-4 上，因此，在这一步骤，我们会将 store-4 和 store-3 上的副本的角色互换，最终 store-3 上的老副本会变成 learner, 而新节点上的副本角色变成 voter.
这个过程为了保证数据的安全性，实际情况分为两条调度指令。
[Image]
具体执行步骤为：
1. PD 通过心跳知道 learner 添加成功后，发送角色互换的指导给 leader 节点
2. Leader 节点在 raft-group 里面广播消息
3. 所有副本收到该心跳后，更新自己的配置。至此，该 raft-group 完成角色互换的配置更新。
这个过程因为只涉及逻辑变更，很快，也很难出问题。

 Step3: Remove old peer
经过步骤2，store-4 上已经有一个 follower,  且已经不会有数据访问到 store-3 上的 learner, 因此我们可以安全的删除这个副本。具体执行步骤如下：
1. PD 发送心跳给 leader, 告知删除 store-3 上的副本
2. Leader 在当前 region 的 raft-group 里面广播这个消息
3. 所有副本在收到这个消息后，更新自己的配置信息，而 store-3 则会将自己这个副本完全删除。
[Image]

PD 上关键日志
以上过程，我们可以从 PD 的日志中看到完整的执行情况：
// create balance-region operator
[2024/07/09 16:25:23.105 +08:00] [INFO] [operator_controller.go:488] ["add operator"] [region-id=142725] [operator="\"balance-region {mv peer: store [1] to [166543141]} (kind:region, region:142725(575, 5), createAt:2024-07-09 16:25:23.105733052 +0800 CST m=+1209042.707881308, startAt:0001-01-01 00:00:00 +0000 UTC, currentStep:0, size:92, steps:[0:{add learner peer 166543143 on store 166543141}, 1:{use joint consensus, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner}, 2:{leave joint state, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner}, 3:{remove peer on store 1}], timeout:[17m0s])\""] [additional-info="{\"sourceScore\":\"817938.92\",\"targetScore\":\"4194.83\"}"]
// Step1: add learner node 
[2024/07/09 16:25:23.105 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="add learner peer 166543143 on store 166543141"] [source=create]
[2024/07/09 16:25:23.107 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=142725] [detail="Add peer:{id:166543143 store_id:166543141 role:Learner }"] [old-confver=5] [new-confver=6]
[2024/07/09 16:25:23.108 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="add learner peer 166543143 on store 166543141"] [source=heartbeat]
// Step2: use joint consensus
[2024/07/09 16:25:24.248 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="use joint consensus, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner"] [source=heartbeat]
[2024/07/09 16:25:24.250 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=142725] [detail="Remove peer:{id:142726 store_id:1 },Remove peer:{id:166543143 store_id:166543141 role:Learner },Add peer:{id:142726 store_id:1 role:DemotingVoter },Add peer:{id:166543143 store_id:166543141 role:IncomingVoter }"] [old-confver=6] [new-confver=8]
// Step3: leave joint state
[2024/07/09 16:25:24.250 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="leave joint state, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner"] [source=heartbeat]
[2024/07/09 16:25:24.252 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=142725] [detail="Remove peer:{id:142726 store_id:1 role:DemotingVoter },Remove peer:{id:166543143 store_id:166543141 role:IncomingVoter },Add peer:{id:142726 store_id:1 role:Learner },Add peer:{id:166543143 store_id:166543141 }"] [old-confver=8] [new-confver=10]
// Step4: remove learner node
[2024/07/09 16:25:24.252 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=142725] [step="remove peer on store 1"] [source=heartbeat]
[2024/07/09 16:25:24.255 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=142725] [detail="Remove peer:{id:142726 store_id:1 role:Learner }"] [old-confver=10] [new-confver=11]

// operator finished
[2024/07/09 16:25:24.255 +08:00] [INFO] [operator_controller.go:635] ["operator finish"] [region-id=142725] [takes=1.149978179s] [operator="\"balance-region {mv peer: store [1] to [166543141]} (kind:region, region:142725(575, 5), createAt:2024-07-09 16:25:23.105733052 +0800 CST m=+1209042.707881308, startAt:2024-07-09 16:25:23.10595495 +0800 CST m=+1209042.708103209, currentStep:4, size:92, steps:[0:{add learner peer 166543143 on store 166543141}, 1:{use joint consensus, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner}, 2:{leave joint state, promote learner peer 166543143 on store 166543141 to voter, demote voter peer 142726 on store 1 to learner}, 3:{remove peer on store 1}], timeout:[17m0s]) finished\""] [additional-info="{\"cancel-reason\":\"\",\"sourceScore\":\"817938.92\",\"targetScore\":\"4194.83\"}"]
绿色的代表：PD 发给 region leader 的信息
红色的代表：region leader 发给 PD 的信息
在具体的运维过程中，当发现上线速度缓慢时，如果有大量的 balance region operator 执行超时，我们可以通过查看 PD 日志中具体某个 operator 的执行情况，对照着以上日志，来看 balance-region 具体卡在哪一步。
实时监控扩容速度
从上文我们知道，扩容 TiKV 的速度主要取决于数据的搬迁情况，主要分为两个因素：
1. PD 产生 balance-region  operator 的速度
2. TiKV 消费 balance-region 的速度，也就是物理数据的搬迁速度。
在扩容后，要做到整个集群的资源的快速均匀，一般跟我们 region 的数量、size 有直接关系，当然与我们集群的繁忙程度也有关系。目前，PD 提供了一些监控可以让我们看到扩容的状态，主要有
- PD->metrics:
  - Region health
    - Pending-region-count: 正准备 add learner 但还没添加成功的副本
    - Learner-peer-count: 当前集群中 learner 副本的个数，需要注意的是，如果集群本身有 tiflash 节点，这个数量也包含了 tiflash 节点里的 learner 个数。
  - Speed：
    - Online store progress：当前扩容进度，根据剩余需要 balance 的空间计算得出
    - Left time: 预估剩余时间，根据剩余需要 balance 的空间及当前数据搬迁速度得出
    - Current scaling speed: 当前扩容数据搬迁实际速度
[Image]

扩容 operator 生成速度
TiDB 集群 Region 的负载均衡调度主要依赖 Balance-region/leader-scheduler, 顾名思义：
- Balance-region- scheduler：负责将 Region 均匀的分散在集群中所有的 store 上，分散存储压力
- Balance-leader-scheduler: 负责将 region leader 均匀分布在 store 上，主要负责分散客户端的请求压力（CPU ）
对于 balance-leader-scheduler, 因为没有数据搬迁，只是 raft-group 元数据的变更，因此特别快。一般情况下，我们不需要特别关注这个（也很少出问题）
本节将重点介绍 balance-region-scheduler, 也就是扩容情况下，迅速往新扩容 kv 上搬迁副本的调度器行为及常见问题。

首先我们来看一下 balance-region-scheduler 是如何选择并生成 balance-region operator 的：
[Image]
Balance-region-scheduler 每隔 10ms~5s/10s 会发起一次调度。
调度顺序
1. 检查 region-schedule-limit, 用于控制当前 balance-region operator 的并发数量。
2. 选择本次要搬走的 store
  1. 过滤选出可以作为数据源搬迁的 store, 过滤条件有：
    1. 检查 store-limit 是否符合条件：store limit remove-peer 
    2. 检查这个 store 上的 max-snapshot-count 是否超过
  1. 在符合要求的 store 里面，选出最适合搬走副本的 store, 按照 region_score 倒叙排序, 优先考虑的条件有：
    1. 空间不足时，优先选剩余空间最不足的节点 （使得 tikv 的剩余数据量均衡）
    2. 空间富裕时，选择已用空间最多的节点（使得 tikv 的数据量分布均衡）
    3. 中间状态综合考虑两个因素
3. 选择要搬走的副本，从当前选中的 source store 中选择一个副本，选择条件按优先级如下：
  1.  pending regions
  2.  followers
  3. Leaders
  4. Learners
4. 选择一个目标 store 作为当前副本的目标
  1. 通过 placement-rule 选择符合要求的 store
  2. 检查 store limit add-peer
  3. 检查 max-snapshot-count 
  4. 检查 max-pending-peer-count
  5. 最后对以上条件都符合的目标 store , 选择 region_score 最小的那个节点

常见问题
scheduler 启用情况
默认情况下 balance-region-scheduler 会被启用，但确实能够用 pd-ctl 将其删除。当发现没有 balance-region operator 生成时，第一步需要检查的便是确认 balance region 是否被启用。
- 通过监控查看 balance-region-scheduler 是否被启用，即调度发生的频率：
  - PD->scheduler->scheduler is running
- 通过 pd-ctl 查看当前正在运行的 schedule
[Image]

- 编辑 balance-region-scheduler
remove balance-region-scheduler
~$ tiup ctl:v7.5.2 pd schedule remove balance-region-scheduler
Starting component ctl: /home/tidb/.tiup/components/ctl/v7.5.2/ctl pd schedule remove balance-region-scheduler Success!
add balance-region-scheduler
~$ tiup ctl:v7.5.2 pd schedule add balance-region-scheduler
Starting component ctl: /home/tidb/.tiup/components/ctl/v7.5.2/ctl pd schedule add balance-region-scheduler Success! The scheduler is created."

检查 region-schedule-limit 
[Image]
Region-schedule-limit 是用来控制 balance-region / scatter-range/hot-region 等 region 相关的 operator 生成的并发度的，默认值是 2048，一般很难到达瓶颈。
- 通过 pd-ctl 检查及设置
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

- 通过监控看是否遇到瓶颈: PD ->operator->schedule reach limit

检查要被 move peer 的 store 
[Image]
一般在 scale out 的过程中，已用空间比较大的 store 很容易被选为 balance-region 的目标对象，因此已用空间比较大的那个 store 很容易受到 store limit remove-peer 的影响。
- 通过监控查看 store limit 配置：PD->cluster->PD scheduler config/store limit
[Image]
- Store limit remove peer 不足的场景：PD->scheduler->filter source 看到大量的 balance-region-XXX-remove-limit 时：
[Image]

检查待 add peer 的 store
[Image]

在扩容场景下，这类 store 往往是新扩容的节点，因此这些新节点很容易变成热点，相关配置也比较容易到达使用瓶颈。
这里相关配置主要是以下三个：
- store limit add-peer : speed limit for the special store
- max-snapshot-count : when the number of snapshots that a single store receives or sends meet the limit, it will never be chosen as a source or target store
- max-pending-peer-count: control the maximum number of pending peers in a single store.  

- Metrics 中查看配置项(同 source store 配置项查看)：pd->cluster->pd scheduler config/store limit
- 查看目标节点配置项是否遇到瓶颈：pd->schedule -> filter target -> balance-regionXXX-{config}-filter

生成 operator 的速度
[Image]
经过以上步骤，一个 balance region 的 operator 变生成了。我们可以通过 pd->operator->schedule operator create -> balance region 查看当前 operator 的产生速度：
Summary 
PD 生成 balance region operator 的速度，直接影响整个扩容的速度。为了保证 PD 在生成 operator 的速度不会成为瓶颈，我们可以根据上文中的监控来确定以下配置项是否设置合理，进行适当的调优：
- ensure balance-region-scheduler is running
- region-schedule-limit ：control the generation speed of schedule-region operator 
- store limit remove-peer/add-peer : speed limit for the special store
- max-snapshot-count : when the number of snapshots that a single store receives or sends meet the limit, it will never be choosed as a source or target store
- max-pending-peer-count: control the maximum number of pending peers in a single store.  

如何判断当前扩容的瓶颈在 TiKV 还是在 PD上
[Image]
我们可以通过对比 PD 上的以上两个监控：
- PD->operators->schedule operator create
- PD->operators->schedule operator finish 
来判断 operator 的消费速度能否跟得上 生成速度，如果不能，说明 TiKV 中出现了瓶颈，则需要继续从 TiKV 中去寻找答案。

TiKV 副本搬迁原理及常见问题
TiKV 之间的副本搬迁一般出现在：
- 扩容新节点，需要将数据在新老 tikv 节点做 balance, 老节点上的数据会往新节点搬迁。在 tikv 数量 VS 新节点数量比较大的情况下，新节点的写入压力最可能成为瓶颈
- 下线节点，下线过程中需要将下线 kv 上的数据搬迁到现有 tikv 中
- 热点调度及正常的 balance-region 调度等
副本搬迁的完整步骤在第一章已经介绍过，下面我们重点介绍一下副本搬迁的 add learner 和 remove learner 操作。
Add learner 步骤概览
[Image]
接下来，我们从 TiKV 中线程池的视角，看一下 add learner 具体是怎么操作的：
[Image]
1. 首先 tikv 由 pd-worker 处理 来自 PD 的 hearbeat, PD 在收到 add learner 请求后，将这个消息交给了 raftstore
2. Raftstore 收到 add learner 操作的请求后，作为 region 的 leader 主要做了以下两件事情：
  1. 在 raft group 里 广播 add learner 的基本信息
    1.  要加 learner 的 tikv 节点也会在本地将这个 region 的元数据创建出来
  2. 交给 apply 线程池 apply 这条 add learner 的 admin 消息
3. Apply 线程池在收到 add learner 这个消息后，更新本地 region 元数据信息后，开始生成 snapshot 并发送给目标 tikv, 这里主要涉及到以下几个线程池：
  1. Region-worker 处理 apply 线程池发过来的 Task::Gen 任务消息，将任务发给 snap-generator 去生成 snapshot.
4. Apply 收到 snap-generator 生成 snapshot 完成的通知后，准备 Task::Send 消息给 snap-handler, snap-handler 处理该任务消息，最终发给 snap-sender 线程池，让其发送 snapshot 给 leaner 节点
5. Learner 节点所在 TiKV 的 GRPC 线程池收到 snapshot 的消息后，准备 Task::REcv 任务给 snap-handler 线程池，snap-hander 处理该任务消息，最终将收 snapshot 的工作交接给 snap-sender 执行。
6. Snap-sender 收取完 snapshot 数据后，通知 apply 线程池 snapshot 文件收集完毕
7. Apply 线程发送给 region-worker, 让其去将 snapshot apply 到 rocksdb 里面。
8. Region-worker 处理 apply-snapshot 消息，让 rocksdb 去 apply snapshot 到本地。

以上步骤中，只要任何一个线程池卡住，都会造成 add learner 操作耗时上升。一般的，当我们发现 add-learner 超时时，可以将涉及数据搬迁的关键节点日志拿出来分析。
Add leaner 时 Leader 所在节点日志示例
Step1: receive AddLeader operator from PD:
// thread-id:33 name: pd-worker
// handle heartbeat from PD:change peer: add learner peer 166543143 store_id: 166543141
// broadcast messages to raft-group
[2024/07/09 16:25:23.106 +08:00] [INFO] [pd.rs:1648] ["try to change peer"] [changes="[peer { id: 166543143 store_id: 166543141 role: Learner } change_type: AddLearnerNode]"] [region_id=142725] [thread_id=33]
Step2: broadcast config change
// thread-id:367 name: raftstore_* ??
[2024/07/09 16:25:23.106 +08:00] [INFO] [peer.rs:4801] ["propose conf change peer"] [kind=Simple] [changes="[change_type: AddLearnerNode peer { id: 166543143 store_id: 166543141 role: Learner }]"] [peer_id=142728] [region_id=142725] [thread_id=367]
// thread-id:369 name: apply-?
[2024/07/09 16:25:23.107 +08:00] [INFO] [apply.rs:1699] ["execute admin command"] [command="cmd_type: ChangePeerV2 change_peer_v2 { changes { change_type: AddLearnerNode peer { id: 166543143 store_id: 166543141 role: Learner } } }"] [index=18] [term=14] [peer_id=142728] [region_id=142725] [thread_id=369]
[2024/07/09 16:25:23.107 +08:00] [INFO] [apply.rs:2293] ["exec ConfChangeV2"] [epoch="conf_ver: 5 version: 575"] [kind=Simple] [peer_id=142728] [region_id=142725] [thread_id=369]
[2024/07/09 16:25:23.107 +08:00] [INFO] [apply.rs:2474] ["conf change successfully"] ["current region"="id: 142725 start_key: …end_key:  region_epoch { conf_ver: 6 version: 575 } peers { id: 142726 store_id: 1 } peers { id: 142727 store_id: 2 } peers { id: 142728 store_id: 3 } peers { id: 166543143 store_id: 166543141 role: Learner }"] ["original region"="id: 142725 start_key: end_key: region_epoch { conf_ver: 5 version: 575 } peers { id: 142726 store_id: 1 } peers { id: 142727 store_id: 2 } peers { id: 142728 store_id: 3 }"] [changes="[change_type: AddLearnerNode peer { id: 166543143 store_id: 166543141 role: Learner }]"] [peer_id=142728] [region_id=142725] [thread_id=369]
[2024/07/09 16:25:23.107 +08:00] [INFO] [raft.rs:2660] ["switched to configuration"] [config="Configuration { voters: Configuration { incoming: Configuration { voters: {142728, 142726, 142727} }, outgoing: Configuration { voters: {} } }, learners: {166543143}, learners_next: {}, auto_leave: false }"] [raft_id=142728] [region_id=142725] [thread_id=367]
[2024/07/09 16:25:23.107 +08:00] [INFO] [peer.rs:4055] ["notify pd with change peer region"] [region="id: 142725 start_key: end_key: region_epoch { conf_ver: 6 version: 575 } peers { id: 142726 store_id: 1 } peers { id: 142727 store_id: 2 } peers { id: 142728 store_id: 3 } peers { id: 166543143 store_id: 166543141 role: Learner }"] [peer_id=142728] [region_id=142725] [thread_id=367]
Step3: prepare snapshot 
// raftstore: process request get_snapshot
[2024/07/09 16:25:23.109 +08:00] [INFO] [peer_storage.rs:549] ["requesting snapshot"] [request_peer=166543143] [request_index=0] [peer_id=142728] [region_id=142725] [thread_id=367]
Step3.1: generate snapshot
// thread-id:349 name: snap-generator thread-num:snap_generator_pool_size
[2024/07/09 16:25:23.111 +08:00] [INFO] [snap.rs:916] ["scan snapshot of one cf"] [size=0] [key_count=0] [cf=default] [snapshot=/DATA/disk2/shirly/tiup/tidb-data/tikv-20161/snap/gen_142725_14_18_(default|lock|write).sst] [region_id=142725] [thread_id=349]
[2024/07/09 16:25:23.111 +08:00] [INFO] [snap.rs:916] ["scan snapshot of one cf"] [size=0] [key_count=0] [cf=lock] [snapshot=/DATA/disk2/shirly/tiup/tidb-data/tikv-20161/snap/gen_142725_14_18_(default|lock|write).sst] [region_id=142725] [thread_id=349]
[2024/07/09 16:25:24.089 +08:00] [INFO] [io.rs:231] ["build_sst_cf_file_list builds 1 files in cf write. Total keys 645299, total size 100663198. raw_size_per_file 104857600, total takes 977.622872ms"] [thread_id=349]
[2024/07/09 16:25:24.097 +08:00] [INFO] [snap.rs:916] ["scan snapshot of one cf"] [size=100663198] [key_count=645299] [cf=write] [snapshot=/DATA/disk2/shirly/tiup/tidb-data/tikv-20161/snap/gen_142725_14_18_(default|lock|write).sst] [region_id=142725] [thread_id=349]
[2024/07/09 16:25:24.097 +08:00] [INFO] [snap.rs:1090] ["scan snapshot"] [takes=987.083966ms] [size=14988542] [key_count=645299] [snapshot=/DATA/disk2/shirly/tiup/tidb-data/tikv-20161/snap/gen_142725_14_18_(default|lock|write).sst] [region_id=142725] [thread_id=349]
Step3.2: send snapshot
thread_id:366 raftstore
[2024/07/09 16:25:24.098 +08:00] [INFO] [peer_storage.rs:510] ["start sending snapshot"] [request_peer=166543143] [peer_id=142728] [region_id=142725] [thread_id=366]
// thread-id:328 name: snap-generator
[2024/07/09 16:25:24.099 +08:00] [INFO] [snap.rs:679] ["set_snapshot_meta total cf files count: 3"] [thread_id=328]
// thread_id:396,name:snap-sender, pool_size: 4(unconfigable)
[2024/07/09 16:25:24.239 +08:00] [INFO] [snap.rs:584] ["sent snapshot"] [duration=137.340853ms] [size=14988542] [snap_key=142725_14_18] [region_id=142725] [thread_id=396]
[2024/07/09 16:25:24.239 +08:00] [INFO] [peer.rs:2024] ["report snapshot status"] [status=Finish] [to="id: 166543143 store_id: 166543141 role: Learner"] [peer_id=142728] [region_id=142725] [thread_id=367]

Add learner 时 Learner 所在节点日志示例
Step1: receive AddLeader raftmessage from leader:create empty peer 
thread_id:362 raftstore:handle raft messages
[2024/07/09 16:25:23.108 +08:00] [INFO] [peer.rs:325] ["replicate peer"] [create_by_peer_store_id=3] [create_by_peer_id=142728] [store_id=166543141] [peer_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:23.108 +08:00] [INFO] [raft.rs:2660] ["switched to configuration"] [config="Configuration { voters: Configuration { incoming: Configuration { voters: {} }, outgoing: Configuration { voters: {} } }, learners: {}, learners_next: {}, auto_leave: false }"] [raft_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:23.108 +08:00] [INFO] [raft.rs:1127] ["became follower at term 0"] [term=0] [raft_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:23.108 +08:00] [INFO] [raft.rs:388] [newRaft] [peers="Configuration { incoming: Configuration { voters: {} }, outgoing: Configuration { voters: {} } }"] ["last term"=0] ["last index"=0] [applied=0] [commit=0] [term=0] [raft_id=166543
143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:23.108 +08:00] [INFO] [raw_node.rs:315] ["RawNode created with id 166543143."] [id=166543143] [raft_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:23.108 +08:00] [INFO] [raft.rs:1362] ["received a message with higher term from 142728"] ["msg type"=MsgHeartbeat] [message_term=14] [term=0] [from=142728] [raft_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:23.108 +08:00] [INFO] [raft.rs:1127] ["became follower at term 14"] [term=14] [raft_id=166543143] [region_id=142725] [thread_id=362]
Step2: receive snapshot file from leader
// thread-id:392 , name:snap_sender
[2024/07/09 16:25:24.224 +08:00] [INFO] [snap.rs:342] ["saving snapshot file"] [file=/DATA/disk1/shirly/tiup/tidb-data/tikv-20164/snap/rev_142725_14_18_(default|lock|write).sst] [snap_key=142725_14_18] [thread_id=392]
[2024/07/09 16:25:24.235 +08:00] [INFO] [snap.rs:350] ["saving all snapshot files"] [takes=134.603179ms] [snap_key=142725_14_18] [thread_id=392]
Step3: restore snapshot file from leader
// thread-id:362,raftstore
[2024/07/09 16:25:24.235 +08:00] [INFO] [raft_log.rs:662] ["log [committed=0, persisted=0, applied=0, unstable.offset=1, unstable.entries.len()=0] starts to restore snapshot [index: 18, term: 14]"] [snapshot_term=14] [snapshot_index=18] [log="committe
d=0, persisted=0, applied=0, unstable.offset=1, unstable.entries.len()=0"] [raft_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:24.235 +08:00] [INFO] [raft.rs:2660] ["switched to configuration"] [config="Configuration { voters: Configuration { incoming: Configuration { voters: {142728, 142726, 142727} }, outgoing: Configuration { voters: {} } }, learners: {16
6543143}, learners_next: {}, auto_leave: false }"] [raft_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:24.235 +08:00] [INFO] [raft.rs:2644] ["restored snapshot"] [snapshot_term=14] [snapshot_index=18] [last_term=14] [last_index=18] [commit=18] [raft_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:24.235 +08:00] [INFO] [raft.rs:2525] ["[commit: 18, term: 14] restored snapshot [index: 18, term: 14]"] [snapshot_term=14] [snapshot_index=18] [commit=18] [term=14] [raft_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:24.235 +08:00] [INFO] [peer_storage.rs:607] ["begin to apply snapshot"] [peer_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:24.235 +08:00] [INFO] [peer_storage.rs:690] ["apply snapshot with state ok"] [for_witness=false] [state="applied_index: 18 truncated_state { index: 18 term: 14 }"] [region="id: 142725 start_key:end_key: region_epoch { conf_ver: 6 version: 575 } peers { id: 142726
store_id: 1 } peers { id: 142727 store_id: 2 } peers { id: 142728 store_id: 3 } peers { id: 166543143 store_id: 166543141 role: Learner }"] [peer_id=166543143] [region_id=142725] [thread_id=362]
[2024/07/09 16:25:24.236 +08:00] [INFO] [peer.rs:5041] ["snapshot is persisted"] [destroy_regions="[]"] [region="id: 142725 start_key: end_key:  region_epoch { conf_ver: 6 version: 575 } peers { id: 142726 store_id: 1 } peers { id: 142727 store_id: 2 } peers { id: 142728 store_id: 3
} peers { id: 166543143 store_id: 166543141 role: Learner }"] [peer_id=166543143] [region_id=142725] [thread_id=362]
// thread-id:341 name: region_worker:1 thread unconfigable
Step4: apply snapshot to rocksdb
[2024/07/09 16:25:24.236 +08:00] [INFO] [region.rs:455] ["begin apply snap data"] [peer_id=166543143] [region_id=142725] [thread_id=341]
[2024/07/09 16:25:24.248 +08:00] [INFO] [region.rs:503] ["apply new data"] [time_takes=11.639116ms] [region_id=142725] [thread_id=341]
[thread_id=367]


常见问题
Add leaner 卡住最常见的原因一般有以下几个：
- CPU 瓶颈：过程中涉及的线程池出现瓶颈
- IO 和网络瓶颈：snapshot 搬迁过程中涉及 io 和 网络开销
- Rocksdb apply snapshot 卡住，往往在 L0 文件堆积较多时，apply snapshot 就会卡住。从对应日志中可以看到最后一步日志迟迟没有出现或者完全卡死。
Add learner 陷入循环
Add learner 卡住后可能导致 tikv 这边进入死循环而导致副本搬迁陷入僵局，以下线为例来说：
1. PD rule checker 执行时，fix-unhealthy peer 时生成 replace-rule-offline(leader)-peer
2. PD 下发 add learner
3. TiKV 开始执行 add learner 
4.  PD 20 分钟后依旧没有收到 add learner 成功的消息，认为该任务超时，取消当前 replace-rule-offline-peer operator 。
5. TiKV 在 30 分钟后  add learner 成功 ，TiKV 上报心跳给 PD，当前这个 region 会有一个 learner 节点。
6. PD rule-cheker 执行时，先看到 orphan-peer, 就把之前添加成功的 learner 删除了
7. PD rule checker 再次执行时，回到步骤 1 , 至此进入循环。
线程池相关问题
region_worker 线程池相关问题
Region worker 主要职责为处理 snapshot generate/apply/destroy 消息
- 线程名：`region-worker`
- Pending-tasks-label:"snapshot-worker"
- 线程数：1（不可配置）
- 处理任务：(Task<EK::Snapshot>)
  - Task::Gen: 生成 snapshot, 会转发给 snap-generator 线程池处理
  - Task::Apply: apply snapshot，会让 rocksdb 将已经收到的 snapshot 文件 apply 到 rocksdb 中。‘
  - Task::Destroy 删除 snapshot, 主要是删除一段连续的范围，也就是删除 leaner 时使用。
- 常见问题：
  - region- worker CPU 成为瓶颈：
    - 现象：
      - 从 CPU 粒度看 region worker CPU 已经跑满（目前没有单独的监控，可以用 tikv->CPU 下面选一个面板编辑公式得到）
[Image]
      - 从 tikv-details -> task -> worker pending tasks->snapshot-worker 的数量会堆积
    - workaround：因为 region worker CPU 只有一个线程且无法编辑，因此这一块没有特别好的方式。主要根据 region worker CPU 跑满的具体原因进行对应的限流：
      - Generate snapshot 压力大：适当将上面需要搬迁的 region 的 leader transfer 走一些。极端情况下可以使用 evict-leader 
      - Apply snapshot 压力大：在 pd 侧用 store limit add-peer 进行限流
      - 删除 snapshot 压力大：在 PD 侧 store limit remove-peer 限流

snap-generator 线程池相关问题
该线程池主要用于生成具体的 snapshot，主要需要注意下面两个瓶颈
- CPU ，线程数：1～16，可通过 snap-generator-pool-size （default 2）调节
- IO 使用限制，通过 snap-io-max-bytes-per-sec（100MB） 控制
因此，当注意到以上两个地方遇到瓶颈时，可以通过修改对应配置项进行调整。如何判断以上两个到达性能瓶颈：
- CPU：与看 region-worker CPU 类似的方式，编辑公式后查看 snap-generator CPU 的使用情况
- IO：通过 tikv-details->snapshot-> snapshot actions/snapshot size 估算得出，或者根据 snapshot transaport speed 估算。
可以通过 online config 直接修改，无需重启 tikv 实例：
// Set snap-io-max-bytes-per-sec online
mysql> set config "127.0.0.1:20165" `server.snap-io-max-bytes-per-sec`= '200MiB';
Query OK, 0 rows affected (0.01 sec)
// Show configs
mysql> show config where type='tikv' and name like '%snap-io-max-bytes-per-sec%';
+------+-----------------+----------------------------------+--------+
| Type | Instance        | Name                             | Value  |
+------+-----------------+----------------------------------+--------+
| tikv | 127.0.0.1:20160 | server.snap-io-max-bytes-per-sec | 100MiB |
| tikv | 127.0.0.1:20162 | server.snap-io-max-bytes-per-sec | 100MiB |
| tikv | 127.0.0.1:20161 | server.snap-io-max-bytes-per-sec | 100MiB |
| tikv | 127.0.0.1:20164 | server.snap-io-max-bytes-per-sec | 100MiB |
| tikv | 127.0.0.1:20165 | server.snap-io-max-bytes-per-sec | 200MiB |
+------+-----------------+----------------------------------+--------+
5 rows in set (0.02 sec)
// Set snap-generator-pool-size online
mysql> set config "127.0.0.1:20165" `raftstore.snap-generator-pool-size`=6;
Query OK, 0 rows affected (0.01 sec)

// Show configs
mysql> show config where type='tikv' and name like '%snap-generator-pool-size%';
+------+-----------------+------------------------------------+-------+
| Type | Instance        | Name                               | Value |
+------+-----------------+------------------------------------+-------+
| tikv | 127.0.0.1:20160 | raftstore.snap-generator-pool-size | 2     |
| tikv | 127.0.0.1:20162 | raftstore.snap-generator-pool-size | 2     |
| tikv | 127.0.0.1:20161 | raftstore.snap-generator-pool-size | 2     |
| tikv | 127.0.0.1:20164 | raftstore.snap-generator-pool-size | 2     |
| tikv | 127.0.0.1:20165 | raftstore.snap-generator-pool-size | 6     |
+------+-----------------+------------------------------------+-------+
5 rows in set (0.02 sec)


snap-handler&snap_sender 线程池相关问题
snap-handler 线程池主要用于分发调度 send/receive snapshot 相关的任务。而真正的发送和接收 snapshot 任务，则由 snap_sender 这个子线程池来完成。
- snap-handler 线程数 1，写死不可编辑
- snap-handler 主要处理的任务有三种类型：
  - Task::Recv — from: kv_service.snap_scheduler
    - process by `snap_sender` pool
  - Task::Send — from: AsyncRaftSender.snap_scheduler 
    - process by `snap_sender` pool 
  - Task::RefreshConfigEvent — from: server.snap_scheduler 
- 控制 snapshot 发送和接收的并发：
  - concurrent-send-snap-limit 32 by default
  - concurrent-recv-snap-limit 32 by default
// set concurrent-send-snap-limit online
mysql> set config "127.0.0.1:20165" `server.concurrent-send-snap-limit`=1;
Query OK, 0 rows affected (0.01 sec)
// Show config
mysql> show config where type='tikv' and name like '%concurrent-send-snap-limit%';
+------+-----------------+-----------------------------------+-------+
| Type | Instance        | Name                              | Value |
+------+-----------------+-----------------------------------+-------+
| tikv | 127.0.0.1:20160 | server.concurrent-send-snap-limit | 32    |
| tikv | 127.0.0.1:20162 | server.concurrent-send-snap-limit | 32    |
| tikv | 127.0.0.1:20161 | server.concurrent-send-snap-limit | 32    |
| tikv | 127.0.0.1:20164 | server.concurrent-send-snap-limit | 32    |
| tikv | 127.0.0.1:20165 | server.concurrent-send-snap-limit | 1     |
+------+-----------------+-----------------------------------+-------+

// set concurrent-recv-snap-limit online
mysql> set config "127.0.0.1:20165" `server.concurrent-recv-snap-limit`=48;
Query OK, 0 rows affected (0.01 sec)

// Show config
mysql> show config where type='tikv' and name like '%concurrent-recv-snap-limit%';
+------+-----------------+-----------------------------------+-------+
| Type | Instance        | Name                              | Value |
+------+-----------------+-----------------------------------+-------+
| tikv | 127.0.0.1:20160 | server.concurrent-recv-snap-limit | 32    |
| tikv | 127.0.0.1:20162 | server.concurrent-recv-snap-limit | 32    |
| tikv | 127.0.0.1:20161 | server.concurrent-recv-snap-limit | 32    |
| tikv | 127.0.0.1:20164 | server.concurrent-recv-snap-limit | 32    |
| tikv | 127.0.0.1:20165 | server.concurrent-recv-snap-limit | 48    |
+------+-----------------+-----------------------------------+-------+
5 rows in set (0.02 sec)

- 子线程池 snap_sender 线程数 4，也是写死在代码里无法配置。
- 相关监控：tikv-details->snapshot->snapshot transport speed & snapshot state count
[Image]
 summary
目前 tikv 侧线程池相关可用参数有且只有下面四个，遇到问题时根据上文所述的常见问题特征进行调整：
online config
default 
thread pool

snap-generator-pool-size
2
`snap-generator`
snap-io-max-bytes-per-sec
100MB
`snap-generator`
concurrent-send-snap-limit
32
`snap_handler`.snap_sender
concurrent-recv-snap-limit
32
`snap_handler`.snap_sender
Ingest sst 导致 Apply wait duration 上涨
- 现象：apply wait duration 升高，但是 apply 不高且 apply CPU 不忙且增大 apply-pool-size 无用。
- 原因： ingest sst 会使用到 rocksdb 的 global mutex  tikv#5911  会导致 apply wait 上涨。从老 tikv 将数据搬迁到新 tikv 时，删除老的 region 走的也是 ingest sst 的方式 tikv#7794。 
- Workaround: 业务高峰期降低数据搬迁的速度，必要时 store limit 调整至 0.000001。目前这方面优化还在进行中。

Scale in TiKV nodes 原理及常见问题
Overview
[Image]
对于 TiKV 来说，缩容和扩容本质上都是在删除或添加 tikv 节点后，进行数据搬迁最后让所有 TiKV 达到数据均衡的目的。
[Image]

如图，我们要下线掉 store-4, 则会将 store-4 的数据搬迁到其他节点上。每个 region 副本的具体搬迁原理与扩容是一样的，只是对于 PD 来说，这类 operator 我们称为 replace-rule-offline-peer 
[Image]

TiKV status on PD
[Image]

TiKV 会定期将自己的状态通过心跳的方式上报给 PD，PD 则根据 TiKV 的状态，产生相应的调度，让整个集群的资源调度能够均衡起来。我们可以使用 pd-ctl 去获取 PD 上 tikv 的详细状态，目前 tikv 的状态主要分为以下几类：
-  Up: 正常情况下
- Offline: 主动发起下线
- Disconnect: 当 PD 20s 没有收到 kv 的心跳后，就会被判断为 disconnect，此时这个 tikv 上的数据还不会被搬走。
  - 只要 tikv 恢复心跳，该 tikv 就会恢复到 up 状态。
  - 需要手动执行下线才会变成 offline 状态
- Down: 超过半小时 PD 没有收到 KV 的心跳，则判断为该状态，
  - 这个状态下的 kv 数据会慢慢被搬走。
  - 需要手动执行下线才会变成 offline 状态
  - 只要 tikv 恢复心跳，该 tikv 状态就立刻变回 up 状态。
- Tombstone：当下线状态下的 TiKV 上的数据被完全搬走后，这个 tikv 就会被安全的删除，此时 PD 会将其变为 tombstone. 一旦变为 tombstone 后，将永远无法恢复。
  - 对于长期 down 且上面没有数据的 tikv, 需要手动将其下线才会变成 tombstone，否则会一直是 Down 状态。

常见问题
TiKV 下线原因判断
从上文我们知道，一旦 TiKV 进入 offline 状态，目标 tikv 上的资源就会很快被释放出来，因为资源变少加上数据搬迁，这个过程中会有一些性能抖动。从上面的 tikv 状态切换我们知道，下线 TiKV 的原因一般有以下两个：
- Tikv 宕机： 在现实场景中，机器宕机是个很常见且高频预期中的问题
- 手动下线：常见运维操作
一般的，我们可以通过 pd 的日志看到具体下线的原因，对于 正常下线的 kv, 我们可以根据以下日志示例看到具体下线的时间：
// Scale in with tiup manually: 
tiup cluster scale-in shirlyv7.5.2 --node "127.0.0.1:20165"
// PD log:
// step1: PD receive DeleteStore by API and set the status of store to `offline`
[2024/07/30 22:31:00.444 +08:00] [INFO] [audit.go:126] ["audit log"] [service-info="{ServiceLabel:DeleteStore, Method:HTTP/1.1/DELETE:/pd/api/v1/store/166543141, Component:anonymous, IP:127.0.0.1, Port:60818, StartTime:2024-07-30 22:31:00 +0800 CST, URLParam:{}, BodyParam:}"]
[2024/07/30 22:31:00.444 +08:00] [WARN] [cluster.go:1516] ["store has been offline"] [store-id=166543141] [store-address=127.0.0.1:20164] [physically-destroyed=false]
// Step2: PD will set store limit remove-peer to unlimited to speed up operator generation.
[2024/07/30 22:31:00.447 +08:00] [INFO] [cluster.go:2627] ["store limit changed"] [store-id=166543141] [type=remove-peer] [rate-per-min=100000000]
// PatrolRegion goroutine notice the offline store and create replace-rule-offline-peer to move data to other stores.
[2024/07/30 22:31:00.447 +08:00] [INFO] [operator_controller.go:488] ["add operator"] [region-id=2848] [operator="\"replace-rule-offline-peer {mv peer: store [166543141] to [1]} (kind:replica,region, region:2848(181, 23), createAt:2024-07-30 22:31:00.447901052 +0800 CST m=+3045380.050049300, startAt:0001-01-01 00:00:00 +0000 UTC, currentStep:0, size:93, steps:[0:{add learner peer 215218780 on store 1}, 1:{use joint consensus, promote learner peer 215218780 on store 1 to voter, demote voter peer 166548760 on store 166543141 to learner}, 2:{leave joint state, promote learner peer 215218780 on store 1 to voter, demote voter peer 166548760 on store 166543141 to learner}, 3:{remove peer on store 166543141}], timeout:[17m0s])\""] [additional-info=]
….
[2024/07/31 00:15:45.497 +08:00] [INFO] [operator_controller.go:635] ["operator finish"] [region-id=144173] [takes=1.88436143s] [operator="\"replace-rule-offline-leader-peer {mv peer: store [166543141] to [1]} (kind:replica,region,leader, region:14417
3(589, 35), createAt:2024-07-31 00:15:43.613338508 +0800 CST m=+3051663.215486756, startAt:2024-07-31 00:15:43.613506495 +0800 CST m=+3051663.215654747, currentStep:5, size:95, steps:[0:{add learner peer 311507110 on store 1}, 1:{transfer leader from 
store 166543141 to store 3}, 2:{use joint consensus, promote learner peer 311507110 on store 1 to voter, demote voter peer 166546715 on store 166543141 to learner}, 3:{leave joint state, promote learner peer 311507110 on store 1 to voter, demote voter
 peer 166546715 on store 166543141 to learner}, 4:{remove peer on store 166543141}], timeout:[18m0s]) finished\""] [additional-info="{\"cancel-reason\":\"\"}"]
// Offline finished once all regions has been migrated to other stores
[2024/07/31 00:15:50.500 +08:00] [WARN] [cluster.go:1625] ["store has been Tombstone"] [store-id=166543141] [store-address=127.0.0.1:20164] [state=Offline] [physically-destroyed=false]
[2024/07/31 00:15:50.503 +08:00] [INFO] [cluster.go:2466] ["store limit removed"] [store-id=166543141]


另外，也可以看 PD 监控来区分下线原因：
[Image]
Abnormal stores:
- Offline stores: 正常主动下线的 tikv 数量
- Down stores: 因失去心跳而被动下线的 tikv
Region health:
- Offline-peer-region-count: 代表着主动下线
- Down-peer-region-count: 代表被动下线

下线 operator 生成速度 
Patrol Region
[Image]
PD 作为整个集群的大脑，时刻关注集群的状态，当集群出现非健康状态时产生新的 operator(调度单元) 指导 tikv 进行修复。针对集群的基本逻辑单元 region, PD 也有一个专门的协程负责检查并生成对应的 operator 指导 tikv 进行自愈。
PD 中负责这部分逻辑的在 checkController 中， 其主要工作为，检查每个 region 的状态，必要时生成 operator. 如
- jonstateChecker: 当有 region 的副本（peer） 处于非正常状态时，生成 operator 加速其变成正常状态
- Splitchecker: 当 region 没有按照 label 或者 rule 进行切分时，切分。
- Rule-checker: 当 Region 不符合当前的副本定义规则(placementrule) 时，生成对应调度, 检查顺序如下：
  - Remove orphan peer: 删除多余的副本
  - Check each rules
    - Add-rule-peer if missing
    - Fix unhealthy（搬迁 offline/down 的 tikv 上的数据）：
      - Fix down peer on down tikv
        - Replace-rule-down-leader-peer
        - replace-rule-down-peer
      - Fix offline peer on offline tikv
        - Replace-rule-offline-leader-peer
        - Replace-rule-offline-peer 
- Merge-checker: 当前 region 过小时，尝试合并。
关于这部分逻辑的详细介绍，可以看 这里
我们在下线 tikv 时，PD 就是在这个协程中的 rue checker 这一步产生对应的 operator 来逐步搬走 tikv 节点上的数据的。

配置及相关监控
- #patrol-region-interval  监控：PD 每隔  #patrol-region-interval(10ms) check 128 个 region, 因此当 region 数量比较大时，会直接影响 check 完所有 region 的速度，因此影响到下线时 operator 生成的速度。
  -  pd->schedule->patrol region time 查看该参数是否生效或需要调整
[Image]
- Rule-checker 配置：默认是开启的，可以通过查看 pd->schedule->Rule checker 查看 rule checker 的执行 OPS
- replica-schedule-limit ：rule checker 生成 operator 的并发数
  - 监控：pd->operator->scheduler reach limit 查看 rule-checker-replica 是否到达过使用限制：
[Image]
- 下面三个用于过滤副本搬迁 store 相关 配置与上线时的用法基本一致：
  - store limit remove-peer/add-peer : speed limit for the special store
  - max-snapshot-count : when the number of snapshots that a single store receives or sends meet the limit, it will never be choosed as a source or target store
  - max-pending-peer-count: control the maximum number of pending peers in a single store.
  - 监控： pd->scheduler->filter target/resource  以 rule-checker 为前缀
[Image]
- 从 PD->operator->scheduler operator create 也可以看到对应 operator 的生成速度：
  - Replace-rule-offline-peer
  - Replace-rule-offline-leader-peer
[Image]
常见问题
下线慢之 replace-rule- XXX- peer operator 生成速度不稳定
- 现象：集群比较大时，下线单个 tikv 时，replace-rule-XXX-peer operator 从 pd->operator->operator 监控中生成速度很不稳定，经常掉 0，从而导致整个下线变慢。
- Root Cause: 因 region 总体数量很高，需要下线的副本很少，扫 region 的大部分时间都不需要产生 operator, 从而导致 replace-rule-XXX-peer operator 的生成速度很不稳定。
- workaround：
  - 通过修改 #patrol-region-interval 加速扫 region 的速度（效果有限）
  - 写脚本使用 pd-ctl 手动给这些要下线的副本添加 operator 是最快的方式（常见）
  - 下线速度对业务影响很小，因此如果不着急释放资源的话可以什么都不做。
replace-rule-XXX-peer 生成速度一直为 0
- 现象：tikv 下线中但是几乎没有任何 operator 生成。 排查 PD 日志，搜索到 schedule  deny
grep deny pd.log
[2023/06/01 23:22:05.326 +00:00] [INFO] [audit.go:126] ["Audit Log"] [service-info="{ServiceLabel:SetRegionLabelRule, Method:HTTP/2.0/POST:/pd/api/v1/config/region-label/rule, Component:anonymous, IP:10.250.8.208, StartTime:2023-06-01 23:22:05 +0000 UTC, URLParam:{}, BodyParam:{\"id\":\"f8c58a8f-9f43-449e-9095-5b635cf2464a\",\"labels\":[{\"key\":\"schedule\",\"value\":\"deny\",\"ttl\":\"5m0s\"}],\"rule_type\":\"key-range\",\"data\":[{\"start_key\":\"7480000000000014ff1a5f720131313131ff31313131ff313131ff3131313131ff3131ff313131346f4cff76ff54320000000000faff01696e636f6d696eff67ff000000000000ff0000f70157616c6cff65740000fd017761ff6c6c65740000fd00fe\",\"end_key\":\"7480000000000014ff1a5f7201756e6b6eff6f776e00fe016f75ff74676f696e67ff00ff00000000000000f7ff0157616c6c657400ff00fd0177616c6c65ff740000fd00000000fc\"}]}}"]
2023-06-01 16:20:25
[2023/06/01 23:20:25.333 +00:00] [INFO] [audit.go:126] ["Audit Log"] [service-info="{ServiceLabel:SetRegionLabelRule, Method:HTTP/2.0/POST:/pd/api/v1/config/region-label/rule, Component:anonymous, IP:10.250.8.208, StartTime:2023-06-01 23:20:25 +0000 UTC, URLParam:{}, BodyParam:{\"id\":\"f8c58a8f-9f43-449e-9095-5b635cf2464a\",\"labels\":[{\"key\":\"schedule\",\"value\":\"deny\",\"ttl\":\"5m0s\"}],\"rule_type\":\"key-range\",\"data\":[{\"start_key\":\"7480000000000014ff1a5f720131313131ff31313131ff313131ff3131313131ff3131ff313131346f4cff76ff54320000000000faff01696e636f6d696eff67ff000000000000ff0000f70157616c6cff65740000fd017761ff6c6c65740000fd00fe\",\"end_key\":\"7480000000000014ff1a5f7201756e6b6eff6f776e00fe016f75ff74676f696e67ff00ff00000000000000f7ff0157616c6c657400ff00fd0177616c6c65ff740000fd00000000fc\"}]}}"]
- Root cause: 有些工具会对相关模块停调度，也就是将指定范围打上 schedule deny 的标签。可以通过日志关键字 deny 排查细节。
- Workaround: 把对应标签删除即可恢复调度。
下线 operator 执行速度
下线 operator 和我们上线情况类似，TiKV 侧的数据搬迁速度（addlearner）会是决定下线和上线的关键因素。 下线时关于 TiKV 侧的副本搬迁常见问题我们可以参考上线类似步骤。接下来，我们简单从 PD 的角度，看一下 operator 的执行速度，帮助我们从 PD 侧来判断下线具体卡在了哪一步。
Replace-rule-offline-peer 
// Create operator by PatrolRegion goroutine
[2024/07/30 22:31:00.447 +08:00] [INFO] [operator_controller.go:488] ["add operator"] [region-id=2848] [operator="\"replace-rule-offline-peer {mv peer: store [166543141] to [1]} (kind:replica,region, region:2848(181, 23), createAt:2024-07-30 22:31:00.447901052 +0800 CST m=+3045380.050049300, startAt:0001-01-01 00:00:00 +0000 UTC, currentStep:0, size:93, steps:[0:{add learner peer 215218780 on store 1}, 1:{use joint consensus, promote learner peer 215218780 on store 1 to voter, demote voter peer 166548760 on store 166543141 to learner}, 2:{leave joint state, promote learner peer 215218780 on store 1 to voter, demote voter peer 166548760 on store 166543141 to learner}, 3:{remove peer on store 166543141}], timeout:[17m0s])\""] [additional-info=]
// Send step1 to region-leader by heartbeat: Add learner
[2024/07/30 22:31:00.448 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=2848] [step="add learner peer 215218780 on store 1"] [source=create]
[2024/07/30 22:31:00.450 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=2848] [detail="Add peer:{id:215218780 store_id:1 role:Learner }"] [old-confver=23] [new-confver=24]
[2024/07/30 22:31:00.450 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=2848] [step="add learner peer 215218780 on store 1"] [source=heartbeat]
// Step2: use joint consensus
[2024/07/30 22:31:02.291 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=2848] [step="use joint consensus, promote learner peer 215218780 on store 1 to voter, demote voter peer 166548760 on store 166543141 to learner"] [source=heartbeat]
[2024/07/30 22:31:02.293 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=2848] [detail="Remove peer:{id:166548760 store_id:166543141 },Remove peer:{id:215218780 store_id:1 role:Learner },Add peer:{id:166548760 store_id:166543141 role:DemotingVoter },Add peer:{id:215218780 store_id:1 role:IncomingVoter }"] [old-confver=24] [new-confver=26]
// Step3: leave joint state
[2024/07/30 22:31:02.293 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=2848] [step="leave joint state, promote learner peer 215218780 on store 1 to voter, demote voter peer 166548760 on store 166543141 to learner"] [source=heartbeat]

[2024/07/30 22:31:02.294 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=2848] [detail="Remove peer:{id:166548760 store_id:166543141 role:DemotingVoter },Remove peer:{id:215218780 store_id:1 role:IncomingVoter },Add peer:{id:166548760 store_id:166543141 role:Learner },Add peer:{id:215218780 store_id:1 }"] [old-confver=26] [new-confver=28]
// Step4: remove learner
[2024/07/30 22:31:02.294 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=2848] [step="remove peer on store 166543141"] [source=heartbeat]
[2024/07/30 22:31:02.295 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=2848] [detail="Remove peer:{id:166548760 store_id:166543141 role:Learner }"] [old-confver=28] [new-confver=29]
// replace-rule-offline-peer finished
[2024/07/30 22:31:02.295 +08:00] [INFO] [operator_controller.go:635] ["operator finish"] [region-id=2848] [takes=1.847735887s] [operator="\"replace-rule-offline-peer {mv peer: store [166543141] to [1]} (kind:replica,region, region:2848(181, 23), createAt:2024-07-30 22:31:00.447901052 +0800 CST m=+3045380.050049300, startAt:2024-07-30 22:31:00.448047345 +0800 CST m=+3045380.050195596, currentStep:4, size:93, steps:[0:{add learner peer 215218780 on store 1}, 1:{use joint consensus, promote learner peer 215218780 on store 1 to voter, demote voter peer 166548760 on store 166543141 to learner}, 2:{leave joint state, promote learner peer 215218780 on store 1 to voter, demote voter peer 166548760 on store 166543141 to learner}, 3:{remove peer on store 166543141}], timeout:[17m0s]) finished\""] [additional-info="{\"cancel-reason\":\"\"}"]

Replace-rule-offline-leader-peer
// create operator by PatrolRegion goroutine
[2024/07/30 22:31:07.041 +08:00] [INFO] [operator_controller.go:488] ["add operator"] [region-id=8337] [operator="\"replace-rule-offline-leader-peer {mv peer: store [166543141] to [1]} (kind:replica,region,leader, region:8337(212, 35), createAt:2024-07-30 22:31:07.041448159 +0800 CST m=+3045386.643596407, startAt:0001-01-01 00:00:00 +0000 UTC, currentStep:0, size:95, steps:[0:{add learner peer 215333671 on store 1}, 1:{transfer leader from store 166543141 to store 3}, 2:{use joint consensus, promote learner peer 215333671 on store 1 to voter, demote voter peer 166546136 on store 166543141 to learner}, 3:{leave joint state, promote learner peer 215333671 on store 1 to voter, demote voter peer 166546136 on store 166543141 to learner}, 4:{remove peer on store 166543141}], timeout:[18m0s])\""] [additional-info=]
// Send step1 to region-leader by heartbeat: Add learner
[2024/07/30 22:31:07.041 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=8337] [step="add learner peer 215333671 on store 1"] [source=create]
[2024/07/30 22:31:07.043 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=8337] [detail="Add peer:{id:215333671 store_id:1 role:Learner }"] [old-confver=35] [new-confver=36]
[2024/07/30 22:31:07.043 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=8337] [step="add learner peer 215333671 on store 1"] [source=heartbeat]
// Step2: transfer leader to a follower
[2024/07/30 22:31:08.880 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=8337] [step="transfer leader from store 166543141 to store 3"] [source=heartbeat]
[2024/07/30 22:31:08.882 +08:00] [INFO] [region.go:762] ["leader changed"] [region-id=8337] [from=166543141] [to=3]
// Step3: use joint state
[2024/07/30 22:31:08.882 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=8337] [step="use joint consensus, promote learner peer 215333671 on store 1 to voter, demote voter peer 166546136 on store 166543141 to learner"] [source=heartbeat]
[2024/07/30 22:31:08.883 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=8337] [detail="Remove peer:{id:166546136 store_id:166543141 },Remove peer:{id:215333671 store_id:1 role:Learner },Add peer:{id:166546136 store_id:166543141 role:DemotingVoter },Add peer:{id:215333671 store_id:1 role:IncomingVoter }"] [old-confver=36] [new-confver=38]
// Step4: leave Joint state
[2024/07/30 22:31:08.883 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=8337] [step="leave joint state, promote learner peer 215333671 on store 1 to voter, demote voter peer 166546136 on store 166543141 to learner"] [source=heartbeat]
[2024/07/30 22:31:08.883 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=8337] [step="leave joint state, promote learner peer 215333671 on store 1 to voter, demote voter peer 166546136 on store 166543141 to learner"] [source=heartbeat]
[2024/07/30 22:31:08.883 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=8337] [step="leave joint state, promote learner peer 215333671 on store 1 to voter, demote voter peer 166546136 on store 166543141 to learner"] [source=heartbeat]
[2024/07/30 22:31:08.884 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=8337] [detail="Remove peer:{id:166546136 store_id:166543141 role:DemotingVoter },Remove peer:{id:215333671 store_id:1 role:IncomingVoter },Add peer:{id:166546136 store_id:166543141 role:Learner },Add peer:{id:215333671 store_id:1 }"] [old-confver=38] [new-confver=40]
// Step5: remove peer on offline store
[2024/07/30 22:31:08.884 +08:00] [INFO] [operator_controller.go:732] ["send schedule command"] [region-id=8337] [step="remove peer on store 166543141"] [source=heartbeat]
[2024/07/30 22:31:08.885 +08:00] [INFO] [region.go:751] ["region ConfVer changed"] [region-id=8337] [detail="Remove peer:{id:166546136 store_id:166543141 role:Learner }"] [old-confver=40] [new-confver=41]
// replace-rule-offline-leader-peer  finished
[2024/07/30 22:31:08.886 +08:00] [INFO] [operator_controller.go:635] ["operator finish"] [region-id=8337] [takes=1.844455946s] [operator="\"replace-rule-offline-leader-peer {mv peer: store [166543141] to [1]} (kind:replica,region,leader, region:8337(212, 35), createAt:2024-07-30 22:31:07.041448159 +0800 CST m=+3045386.643596407, startAt:2024-07-30 22:31:07.041535502 +0800 CST m=+3045386.643683754, currentStep:5, size:95, steps:[0:{add learner peer 215333671 on store 1}, 1:{transfer leader from store 166543141 to store 3}, 2:{use joint consensus, promote learner peer 215333671 on store 1 to voter, demote voter peer 166546136 on store 166543141 to learner}, 3:{leave joint state, promote learner peer 215333671 on store 1 to voter, demote voter peer 166546136 on store 166543141 to learner}, 4:{remove peer on store 166543141}], timeout:[18m0s]) finished\""] [additional-info="{\"cancel-reason\":\"\"}"]

常见问题
上下线时 PD 上所有 region 相关请求都被卡住 pd#7248
- 现象：集群很大的情况下，上下线时，PD 上所有 region 相关的请求延迟升高，包括：region heartbeat 处理，TiDB 来 get region 等。
[Image]
- Root cause: PD 在估算上下线的剩余时间时，需要通过 GetRegionSizeByRange 去拿剩余 region 空间时共享了管理所有 region 元数据的大锁，如果这个 region 量比较大的情况下，这个过程会很慢，锁住时间就会久。
- Workaround: 
  - 升级到 v6.5.6/7.1.3/7.5.0
  - 设置 max-store-preparing-time 为 10s, 也就是 10 s 后不再估算 上下线进度。

下线速度配置 summary
config
description
componment
replica-schedule-limit
Control the number of tasks scheduling the replica at the same time
PD
max-snapshot-count/max-pending-peer-count
control the maximum number of snapshots that a single store receives or sends at the same time
PD
store limit remove-peer/ add-peer 
speed limit for the special store
PD
snap-max-write-bytes-per-sec
the maximum allowable disk bandwidth when processing snapshots
TiKV
concurrnet-send-snap-limit /concurrent-recv-snap-limit
The maximum number of snapshots send/received at the same time
TiKV
snap-generator-pool-size
The concurrency to generate snapshot
TiKV
