---
layout: post
title: "TiDB 缩容原理及常见问题"
keywords: ["tidb"]
description: "TiDB 缩容原理及常见问题"
category: "tidb"
tags: ["tidb","tikv","pd"]
comments: true
---



作为一个分布式数据库，扩缩容是 TiDB 集群最常见的运维操作之一。本系列文章，我们将基于 v7.5.0 具体介绍扩缩容操作的具体原理、相关配置及常见问题的排查。
说到扩缩容，从集群视角考虑，主要需要考虑的是扩缩容完成后，集群数据通过调度，重新让所有在线的 tikv 的资源使用到达一个平衡的状态。
因此对于扩缩容来说，我们主要关心的还是以下两点：
- 调度产生原理
- 调度执行原理

本系列文章我们将继续围绕以上两个逻辑，重点介绍扩缩容过程中的核心模块及常见问题，分为以下几个部分：
- [TiDB 扩容原理及常见问题](https://andremouche.github.io/tidb/tidb-scale-in-and-out.html)
- [扩容过程中调度生成原理及常见问题](https://andremouche.github.io/tidb/tidb-scale-out-pd.html)
- [缩容过程中调度生成原理及常见问题](https://andremouche.github.io/tidb/tidb-scale-in.html)
- [扩缩容过程调度执行（TiKV 副本搬迁）的原理及常见问题](https://andremouche.github.io/tidb/tidb-move-region-between-stores.html)

本文我们重点介绍 TiDB 缩容原理及常见问题，将侧重于从 PD 视角介绍，也就是调度的生成原理部分，对于缩容过程中调度消费的原理及常见问题，可以看[这篇](https://andremouche.github.io/tidb/tidb-move-region-between-stores.html)文章。

# Overview

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale_overview.png?raw=true" width="600" />

对于 TiKV 来说，缩容和扩容本质上都是在删除或添加 tikv 节点后，进行数据搬迁最后让所有 TiKV 达到数据均衡的目的。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/offline_store.png?raw=true" width="600" />


如图，我们要下线掉 store-4, 则会将 store-4 的数据搬迁到其他节点上。每个 region 副本的具体搬迁原理与扩容是一样的，只是对于 PD 来说，这类 operator 我们称为 `replace-rule-offline-peer` 


<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/replace_rule_offline_peer.png?raw=true" width="600" />


# TiKV status on PD

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/store_status.png?raw=true" width="600" />

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

## 常见问题

## TiKV 下线原因判断

从上文我们知道，一旦 TiKV 进入 offline 状态，目标 tikv 上的资源就会很快被释放出来，因为资源变少加上数据搬迁，这个过程中会有一些性能抖动。从上面的 tikv 状态切换我们知道，下线 TiKV 的原因一般有以下两个：

- Tikv 宕机，变成 down： 在现实场景中，机器宕机是个很常见且高频预期中的问题
- 手动下线,  变成 offline：常见运维操作

一般的，我们可以通过 pd 的日志看到具体下线的原因，对于 正常下线的 kv, 我们可以根据以下日志示例看到具体下线的时间：

``` 
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

```

另外，也可以看 PD 监控来区分下线原因：

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/abnormal_stores.png?raw=true" width="600" />

Abnormal stores:
- Offline stores: 正常主动下线的 tikv 数量
- Down stores: 因失去心跳而被动下线的 tikv
Region health:
- Offline-peer-region-count: 代表着主动下线
- Down-peer-region-count: 代表被动下线

# 下线 operator 生成速度 

## Patrol Region

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/check_region.png?raw=true" width="600" />

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

关于这部分逻辑的详细介绍，可以看另一篇文章。
我们在下线 tikv 时，PD 就是在这个协程中的 rue checker 这一步产生对应的 operator 来逐步搬走 tikv 节点上的数据的。

## 配置及相关监控

- #[patrol-region-interval](https://docs.pingcap.com/zh/tidb/stable/pd-configuration-file#patrol-region-interval) 监控：PD 每隔  #[patrol-region-interval](https://docs.pingcap.com/zh/tidb/stable/pd-configuration-file#patrol-region-interval)(10ms by default) check 128 个 region, 因此当 region 数量比较大时，会直接影响 check 完所有 region 的速度，因此影响到下线时 operator 生成的速度。
  -  pd->schedule->patrol region time 查看该参数是否生效或需要调整


<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/patrol_region.png?raw=true" width="600" />
- Rule-checker 配置：默认是开启的，可以通过查看 pd->schedule->Rule checker 查看 rule checker 的执行 OPS
- [replica-schedule-limit](https://docs.pingcap.com/tidb/dev/pd-configuration-file#replica-schedule-limit) ：rule checker 生成 operator 的并发数
  - 监控：pd->operator->scheduler reach limit 查看 rule-checker-replica 是否到达过使用限制：

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/region_schedule_limit1.png?raw=true" width="600" />

- 下面三个用于过滤副本搬迁 store 相关 配置与上线时的用法基本一致：
  - [store limit remove-peer/add-peer](https://docs.pingcap.com/tidb/stable/configure-store-limit#principles-of-store-limit-v2) : speed limit for the special store
  - [max-snapshot-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-snapshot-count) : when the number of snapshots that a single store receives or sends meet the limit, it will never be choosed as a source or target store
  - [max-pending-peer-count](https://docs.pingcap.com/tidb/stable/pd-configuration-file#max-pending-peer-count): control the maximum number of pending peers in a single store.
  - 监控： pd->scheduler->filter target/resource  以 `rule-checker` 为前缀

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale_in_filter_target.png?raw=true" width="600" />

- 从 PD->operator->scheduler operator create 也可以看到对应 operator 的生成速度：
  - Replace-rule-offline-peer
  - Replace-rule-offline-leader-peer

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/offline_create_operator.png?raw=true" width="600" />

## 常见问题

### 下线慢之 replace-rule- XXX- peer operator 生成速度不稳定

- 现象：集群比较大时，下线单个 tikv 时，replace-rule-XXX-peer operator 从 pd->operator->operator 监控中生成速度很不稳定，经常掉 0，从而导致整个下线变慢。
- Root Cause: 因 region 总体数量很高，需要下线的副本很少，扫 region 的大部分时间都不需要产生 operator, 从而导致 replace-rule-XXX-peer operator 的生成速度很不稳定。
- workaround：
  - 通过修改 [#patrol-region-interval](https://docs.pingcap.com/zh/tidb/stable/pd-configuration-file#patrol-region-interval) 加速扫 region 的速度（效果有限）
  - 写脚本使用 pd-ctl 手动给这些要下线的副本添加 operator 是最快的方式（常见）
  - 下线速度对业务影响很小，因此如果不着急释放资源的话可以什么都不做。

### replace-rule-XXX-peer 生成速度一直为 0

- 现象：tikv 下线中但是几乎没有任何 operator 生成。 排查 PD 日志，搜索到 schedule  deny

``` 
grep deny pd.log
[2023/06/01 23:22:05.326 +00:00] [INFO] [audit.go:126] ["Audit Log"] [service-info="{ServiceLabel:SetRegionLabelRule, Method:HTTP/2.0/POST:/pd/api/v1/config/region-label/rule, Component:anonymous, IP:10.250.8.208, StartTime:2023-06-01 23:22:05 +0000 UTC, URLParam:{}, BodyParam:{\"id\":\"f8c58a8f-9f43-449e-9095-5b635cf2464a\",\"labels\":[{\"key\":\"schedule\",\"value\":\"deny\",\"ttl\":\"5m0s\"}],\"rule_type\":\"key-range\",\"data\":[{\"start_key\":\"7480000000000014ff1a5f720131313131ff31313131ff313131ff3131313131ff3131ff313131346f4cff76ff54320000000000faff01696e636f6d696eff67ff000000000000ff0000f70157616c6cff65740000fd017761ff6c6c65740000fd00fe\",\"end_key\":\"7480000000000014ff1a5f7201756e6b6eff6f776e00fe016f75ff74676f696e67ff00ff00000000000000f7ff0157616c6c657400ff00fd0177616c6c65ff740000fd00000000fc\"}]}}"]
2023-06-01 16:20:25
[2023/06/01 23:20:25.333 +00:00] [INFO] [audit.go:126] ["Audit Log"] [service-info="{ServiceLabel:SetRegionLabelRule, Method:HTTP/2.0/POST:/pd/api/v1/config/region-label/rule, Component:anonymous, IP:10.250.8.208, StartTime:2023-06-01 23:20:25 +0000 UTC, URLParam:{}, BodyParam:{\"id\":\"f8c58a8f-9f43-449e-9095-5b635cf2464a\",\"labels\":[{\"key\":\"schedule\",\"value\":\"deny\",\"ttl\":\"5m0s\"}],\"rule_type\":\"key-range\",\"data\":[{\"start_key\":\"7480000000000014ff1a5f720131313131ff31313131ff313131ff3131313131ff3131ff313131346f4cff76ff54320000000000faff01696e636f6d696eff67ff000000000000ff0000f70157616c6cff65740000fd017761ff6c6c65740000fd00fe\",\"end_key\":\"7480000000000014ff1a5f7201756e6b6eff6f776e00fe016f75ff74676f696e67ff00ff00000000000000f7ff0157616c6c657400ff00fd0177616c6c65ff740000fd00000000fc\"}]}}"]
``` 

- Root cause: 有些工具会对相关模块停调度，也就是将指定范围打上 schedule deny 的标签。可以通过日志关键字 deny 排查细节。
- Workaround: 把对应标签删除即可恢复调度。

# 下线 operator 执行速度

下线 operator 和我们上线情况类似，TiKV 侧的数据搬迁速度（addlearner）会是决定下线和上线的关键因素。 下线时关于 TiKV 侧的副本搬迁常见问题我们可以参考上线类似步骤。接下来，我们简单从 PD 的角度，看一下 operator 的执行速度，帮助我们从 PD 侧来判断下线具体卡在了哪一步。
```
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

``` 

# 常见问题

## 上下线时 PD 上所有 region 相关请求都被卡住 [pd#7248](https://github.com/tikv/pd/issues/7248)

- 现象：集群很大的情况下，上下线时，PD 上所有 region 相关的请求延迟升高，包括：region heartbeat 处理，TiDB 来 get region 等。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/scale_in_status.png?raw=true" width="600" />

- Root cause: PD 在估算上下线的剩余时间时，需要通过 GetRegionSizeByRange 去拿剩余 region 空间时共享了管理所有 region 元数据的大锁，如果这个 region 量比较大的情况下，这个过程会很慢，锁住时间就会久。
- Workaround: 
  - 升级到 v6.5.6/7.1.3/7.5.0
  - 设置 max-store-preparing-time 为 10s, 也就是 10 s 后不再估算 上下线进度。

# 下线速度配置 summary

- [replica-schedule-limit](https://docs.pingcap.com/tidb/dev/pd-configuration-file#replica-schedule-limit): Control the number of tasks scheduling the replica at the same time. [PD]
- [max-snapshot-count/max-pending-peer-count](https://docs.pingcap.com/tidb/dev/pd-configuration-file#max-snapshot-count): control the maximum number of snapshots that a single store receives or sends at the same time. [PD]
- [store limit remove-peer/ add-peer](https://docs.pingcap.com/tidb/stable/configure-store-limit#principles-of-store-limit-v2): speed limit for the special store. [PD]
- [snap-max-write-bytes-per-sec](https://docs.pingcap.com/tidb/v6.5/tikv-configuration-file#snap-max-write-bytes-per-sec): the maximum allowable disk bandwidth when processing snapshots. [TiKV]
- [concurrnet-send-snap-limit /concurrent-recv-snap-limit](https://docs.pingcap.com/tidb/dev/tikv-configuration-file#concurrent-send-snap-limit): The maximum number of snapshots send/received at the same time.[TiKV]
- [snap-generator-pool-size](https://docs.pingcap.com/tidb/dev/tikv-configuration-file#snap-generator-pool-size-new-in-v540): The concurrency to generate snapshot.[TiKV]
