---
layout: post
title: "TiKV 副本搬迁原理及常见问题"
keywords: ["tidb"]
description: "TiKV 副本搬迁原理及常见问题"
category: "tidb"
tags: ["tidb","tikv"]
comments: true
---

# TiKV 副本搬迁原理及常见问题

TiKV 之间的副本搬迁一般出现在：

- 扩容新节点，需要将数据在新老 tikv 节点做 balance, 老节点上的数据会往新节点搬迁。在 tikv 数量 VS 新节点数量比较大的情况下，新节点的写入压力最可能成为瓶颈
- 下线节点，下线过程中需要将下线 kv 上的数据搬迁到现有 tikv 中
- 热点调度及正常的 balance-region 调度等

副本搬迁的完整步骤在[第一章]()已经介绍过，下面我们重点介绍一下副本搬迁的 add learner 和 remove learner 操作。

## Add learner 步骤概览

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/tikv_add_learner.png?raw=true" width="600" />

接下来，我们从 TiKV 中线程池的视角，看一下 add learner 具体是怎么操作的：

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/tikv_add_learner_threadpool?raw=true" width="600" />

1. 首先 tikv 由 `pd-worker` 处理 来自 PD 的 hearbeat, PD 在收到 add learner 请求后，将这个消息交给了 `raftstore`
2. Raftstore 收到 add learner 操作的请求后，作为 region 的 leader 主要做了以下两件事情：
  - 在 raft group 里 广播 add learner 的基本信息
    - 要加 learner 的 tikv 节点也会在本地将这个 region 的元数据创建出来
  - 交给 apply 线程池 apply 这条 add learner 的 admin 消息
3. Apply 线程池在收到 add learner 这个消息后，更新本地 region 元数据信息后，开始生成 snapshot 并发送给目标 tikv, 这里主要涉及到以下几个线程池：
  - `region-worker` 处理 apply 线程池发过来的 `Task::Gen` 任务消息，将任务发给 `snap-generator` 去生成 snapshot.
4. Apply 收到 `snap-generator` 生成 snapshot 完成的通知后，准备 `Task::Send` 消息给 `snap-handler`, `snap-handler` 处理该任务消息，最终发给 `snap-sender` 线程池，让其发送 snapshot 给 leaner 节点
5. Learner 节点所在 TiKV 的 GRPC 线程池收到 snapshot 的消息后，准备 `Task::REcv` 任务给 `snap-handler` 线程池，`snap-hander` 处理该任务消息，最终将收 snapshot 的工作交接给 `snap-sender` 执行。
6. Snap-sender 收取完 snapshot 数据后，通知 apply 线程池 snapshot 文件收集完毕
7. Apply 线程发送给 `region-worker`, 让其去将 snapshot apply 到 rocksdb 里面。
8. `region-worker` 处理 apply-snapshot 消息，让 rocksdb 去 apply snapshot 到本地。

以上步骤中，只要任何一个线程池卡住，都会造成 add learner 操作耗时上升。一般的，当我们发现 add-learner 超时时，可以将涉及数据搬迁的关键节点日志拿出来分析。

## Add leaner 时 Leader 所在节点日志示例

```
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
``` 

## Add learner 时 Learner 所在节点日志示例

```
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
``` 

## 常见问题

Add leaner 卡住最常见的原因一般有以下几个：
- CPU 瓶颈：过程中涉及的线程池出现瓶颈
- IO 和网络瓶颈：snapshot 搬迁过程中涉及 io 和 网络开销
- Rocksdb apply snapshot 卡住，往往在 L0 文件堆积较多时，apply snapshot 就会卡住。从对应日志中可以看到最后一步日志迟迟没有出现或者完全卡死。

### Add learner 陷入循环

Add learner 卡住后可能导致 tikv 这边进入死循环而导致副本搬迁陷入僵局，以下线为例来说：
1. PD rule checker 执行时，fix-unhealthy peer 时生成 replace-rule-offline(leader)-peer
2. PD 下发 add learner
3. TiKV 开始执行 add learner 
4.  PD 20 分钟后依旧没有收到 add learner 成功的消息，认为该任务超时，取消当前 replace-rule-offline-peer operator 。
5. TiKV 在 30 分钟后  add learner 成功 ，TiKV 上报心跳给 PD，当前这个 region 会有一个 learner 节点。
6. PD rule-cheker 执行时，先看到 orphan-peer, 就把之前添加成功的 learner 删除了
7. PD rule checker 再次执行时，回到步骤 1 , 至此进入循环。

### 线程池相关问题

#### `region_worker` 线程池相关问题

Region worker 主要职责为处理 snapshot generate/apply/destroy 消息
- 线程名：`region-worker`
- Pending-tasks-label:"snapshot-worker"
- 线程数：1（不可配置）
- 处理任务：(Task<EK::Snapshot>)
  - `Task::Gen`: 生成 snapshot, 会转发给 `snap-generator` 线程池处理
  - `Task::Apply`: apply snapshot，会让 rocksdb 将已经收到的 snapshot 文件 apply 到 rocksdb 中。‘
  - `Task::Destroy` 删除 snapshot, 主要是删除一段连续的范围，也就是删除 leaner 时使用。
- 常见问题：
  - region-worker CPU 成为瓶颈：
    - 现象：
      - 从 CPU 粒度看 region worker CPU 已经跑满（目前没有单独的监控，可以用 tikv->CPU 下面选一个面板编辑公式得到）
<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/region_worker_cpu.png?raw=true" width="600" />
      - 从 tikv-details -> task -> worker pending tasks->snapshot-worker 的数量会堆积
    - workaround：因为 region worker CPU 只有一个线程且无法编辑，因此这一块没有特别好的方式。主要根据 region worker CPU 跑满的具体原因进行对应的限流：
      - Generate snapshot 压力大：适当将上面需要搬迁的 region 的 leader transfer 走一些。极端情况下可以使用 evict-leader 
      - Apply snapshot 压力大：在 pd 侧用 store limit add-peer 进行限流
      - 删除 snapshot 压力大：在 PD 侧 store limit remove-peer 限流

#### `snap-generator` 线程池相关问题

该线程池主要用于生成具体的 snapshot，主要需要注意下面两个瓶颈
- CPU ，线程数：1～16，可通过 [snap-generator-pool-size](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#snap-generator-pool-size-new-in-v540) （default 2）调节
- IO 使用限制，通过 [snap-io-max-bytes-per-sec](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#snap-io-max-bytes-per-sec)（100MB） 控制

因此，当注意到以上两个地方遇到瓶颈时，可以通过修改对应配置项进行调整。如何判断以上两个到达性能瓶颈：
- CPU：与看 region-worker CPU 类似的方式，编辑公式后查看 snap-generator CPU 的使用情况
- IO：通过 tikv-details->snapshot-> snapshot actions/snapshot size 估算得出，或者根据 snapshot transaport speed 估算。

可以通过 online config 直接修改，无需重启 tikv 实例：

```
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
``` 

#### `snap-handler`&`snap_sender` 线程池相关问题

`snap-handler` 线程池主要用于分发调度 send/receive snapshot 相关的任务。而真正的发送和接收 snapshot 任务，则由 `snap_sender` 这个子线程池来完成。

- snap-handler 线程数 1，写死不可编辑
- snap-handler 主要处理的任务有三种类型：
  - `Task::Recv` — from: kv_service.snap_scheduler
    - process by `snap_sender` pool
  - `Task::Send` — from: AsyncRaftSender.snap_scheduler 
    - process by `snap_sender` pool 
  - `Task::RefreshConfigEvent` — from: server.snap_scheduler 
- 控制 snapshot 发送和接收的并发：
  - [concurrent-send-snap-limit](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#concurrent-send-snap-limit) 32 by default
  - [concurrent-recv-snap-limit](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#concurrent-recv-snap-limit) 32 by default

``` 
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
```

- 子线程池 snap_sender 线程数 4，也是写死在代码里无法配置。
- 相关监控：tikv-details->snapshot->snapshot transport speed & snapshot state count
<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_scale/snap_transport_speed.png?raw=true" width="600" />

### summary
目前 tikv 侧线程池相关可用参数有且只有下面四个，遇到问题时根据上文所述的常见问题特征进行调整：
- [snap-generator-pool-size](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#snap-generator-pool-size-new-in-v540) (default 2), used by `snap-generator`
-  [snap-io-max-bytes-per-sec](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#snap-io-max-bytes-per-sec) (100MB by default),used by `snap-generator`
- [concurrent-send-snap-limit](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#concurrent-send-snap-limit)(32 by default) used by `snap_handler`.`snap_sender`
- [concurrent-recv-snap-limit](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#concurrent-recv-snap-limit) (32 by default) used by `snap_handler`.`snap_sender`

### Ingest sst 导致 Apply wait duration 上涨
- 现象：apply wait duration 升高，但是 apply 不高且 apply CPU 不忙且增大 apply-pool-size 无用。
- 原因： ingest sst 会使用到 rocksdb 的 global mutex  [tikv#5911](https://github.com/tikv/tikv/issues/5911)  会导致 apply wait 上涨。从老 tikv 将数据搬迁到新 tikv 时，删除老的 region 走的也是 ingest sst 的方式 [tikv#7794](https://github.com/tikv/tikv/pull/7794)。 
- Workaround: 业务高峰期降低数据搬迁的速度，必要时 store limit 调整至 0.000001。目前这方面优化还在进行中。

