---
layout: post
title: "[Tidb]Tikv-storage源码阅读"
keywords: ["tidb"]
description: "[Tikv] tikv-storage源码阅读"
category: "tidb"
tags: ["tidb"]
comments: true
---

## Tikv-storage 层源码阅读

项目地址: [github/pingcap/tikv](https://github.com/pingcap/tikv)

## 目录


* Tikv架构
* 内存锁模型
* Column Families
* 整体处理逻辑说明





## TIKV 架构



<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master//images/tidb/tikv_storage.jpg?raw=true" alt="tikv_storage_1" title="tikv_storage.jpg" width="600" />




其中Storage对应目录为tikv/src/storage

其目录结构为

```
├── config.rs
├── engine
│   ├── metrics.rs
│   ├── mod.rs
│   ├── raftkv.rs
│   └── rocksdb.rs
├── metrics.rs
├── mod.rs
├── mvcc
│   ├── lock.rs
│   ├── metrics.rs
│   ├── mod.rs
│   ├── reader.rs
│   ├── txn.rs
│   └── write.rs
├── txn
│   ├── latch.rs
│   ├── mod.rs
│   ├── scheduler.rs
│   └── store.rs
└── types.rs

```

入口为storage/mode.rs的Storage

```
pub struct Storage {
   engine: Box<Engine>,
   sendch: SendCh<Msg>,
   handle: Arc<Mutex<StorageHandle>>,
}
```

* engine指向storage/engine中的engine实现，即指向一个key－value的存储引擎
* sendch为实际处理时，发送消息使用的channel
* handle为实际处理请求的处理入口

### Engine

  engine/raftkv--基于raft的kv实现，下面接的是raftstore,可以看作是一个分布式存储引擎，生产环境中，主要使用该引擎。对raftstore而言，其相当于实现了客户端。对上层而言，是一个分布式的存储引擎。
engine/rocksdb--单机rocksdb的实现，下面接的是rocksdb。对rocksdb而言，其封装实现了客户端。对上层而言，它就是一个单机存储引擎。

### Txn

* txn/scheduler 实现了对客户端传来的各个命令的分发，调度，处理。
* txn/store  对snapshot的封装
* txn/latch 单机内存锁的封装，用以保证在当前机器上，并发场景下，对key操作的一致性
 
### MVCC

* mvcc/txn 封装了多版本读写操作
* mvcc/reader 主要封装了多版本读操作的支持的一些func
* mvcc/write 主要封装了column family write涉及到的逻辑
* mvcc/lock 分布式事务涉及到的lock的抽象，主要为逻辑到具体column familty lock的逻辑处理
* mvcc/metrics:监控相关指标
   

### 内存锁模型 

用于保证单机上对同一个key操作的一致性
对应代码：（/src/storage/txn/latch.rs）acquire

#### 模型

对每个key抽象一个等待队列。
只针对写操作，对于读操作不做限制。
当一个写请求过来时：

1. 尝试获取涉及到的所有key的锁。
2. 若某个key的队列非空，或者队首不是当前操作，则当前操作放入该key的队列并退出。
3. 事务操作完毕时，将当前事务涉及所有的key释放（即当前事务从这些队列中出队），同时，依次唤醒所有队首事务。

##### 示例

假设当前有三个事务依次打过来，T1（key1）,T2(key1,key2,key3),T3(key1,key2,key5)

1. 初始化状态下，所有key(key1, key2, key3, key4, key...)的等待锁队列都为空<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_storage_latch_1.jpg?raw=true" alt="tikv_storage_latch_1" title="tikv_storage_latch_1.jpg" width="600" />
2. T1进来,T1进入key1队列，作为队首，获取key的锁。T1的所有锁获取完毕，开始执行任务<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_storage_latch_2.jpg?raw=true" alt="tikv_storage_latch_2" title="tikv_storage_latch_2.jpg" width="600" />
3. T2 进来，T2进入key1的队列，检测队首不是自己，等待
4. T3 进来，T3进入key1的队列，发现当前队首不是自己，等待<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_storage_latch_3.jpg?raw=true" alt="tikv_storage_latch_3" title="tikv_storage_latch_3.jpg" width="600" />
5. T1 任务结束，从key1队列中去除，发现当前队首为T2,唤醒T2
6. T2 检测当前key1队列队首，发现是自己，获取该锁。尝试获取key2的锁，先进入队列，发现队列是自己，获取key2的锁。尝试获取key3的锁，同样获取key的锁。<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_storage_latch_4.jpg?raw=true" alt="tikv_storage_latch_4" title="tikv_storage_latch_4.jpg" width="600" />
7. T2 的锁获取完毕，开始执行任务
8. T2完成，释放所有锁，并依次唤醒各个锁的等待队列的第一个事务。这里只有一个事务在等待－T3。 T3被唤醒，尝试获取锁，执行，释放。<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_storage_latch_5.jpg?raw=true" alt="tikv_storage_latch_5" title="tikv_storage_latch_5.jpg" width="600" />



## Column Families

一个column Family可以理解为数据库中的一张表。逻辑上同一column中的数据会被存放在一起，独立的compaction,独立存放在sst中。这里我们一共有四个column families，具体如下：

* default 具体数据，结构为（key_ts=>value）
* lock －－存取事务中的锁信息,结构为 (key=>ts,lock_type)
* write －－存事务的commit信息，结构为key_ts=>ts in default
* raft－－raft kv 的log等信息

### default

* default为实际存的数据
* 数据结构为(${key}_${ts}=>${value}), 其中key为实际key,ts为更新时间戳标识(start_ts)，与write中的值对应。value为具体的数据值
 
### lock
 
 * lock为事务锁相关的数据
 * 数据结构为${key}=>(${ts}, ${lock_type}), 其中ts为锁开始时间，lock_type为锁的类型。
 * lock_type目前有三种：PUT／DELETE／Lock

 
### write

* write为数据提交记录
* 数据结构为：${key}_${commit_ts}=>(${value_ts},${type}) 。其中key为实际key, commit_ts为提交时间戳，value_ts为该数据对应的defalut里面的ts; type为提交类型
* 当前type提交类型有：Put／Delete / Lock / Rollback


### 读操作示例

假设当前各family里的数据如下<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_storage_cf_1.jpg?raw=true" alt="tikv_storage_cf_1" title="tikv_storage_cf_1.jpg" width="600" />


#### 数据存在时
读一条数据时，get(key)  ts=9时

1. 从lock中，获取到当前key的锁信息，发现锁的ts为13,大于9，数据可获取
2. 从write中获取比ts＝9小的最近key，这时发现key_6=>5，当前数据类型为PUT，数据可用
3. 从default中获取key_5=>v5 即为所得值

#### 读取数据不存在时

 get(key) ts=4时：

1. 从lock中，获取到当前key的锁信息，发现锁的ts为13,大于4，数据可获取
2. 从write中获取比ts＝4小的最近key，这时发现key_3=>(,delete)，当前数据类型为DELETE
返回数据空，即当前数据不存在

#### 读取数据被锁住时

读取get(key,14)

1. 从lock中，获取到当前key的锁信息，发现锁的ts为13,小于等于14，数据不可获取，等待及重试


### 写操作说明

插入一条数据时（put(key,v)）ts，步骤如下

##### prewrite:

1. 从lock中获取key的锁，如果锁存在且不是当前prewrite的锁，返回当前事务冲突
2. 在lock中为当前事务的key设置lock,如果当前value较短，则将该值存于lock中，返回
3. 若当前value较长，则将当前key_ts＝》value存入default Column Family中

#### commit:

1. 检查当前事务的lock是否存在，若不存在，则返回错误
2. 将当前key,commit_ts=>start_ts存入write 所在CF
3. 从lock CF 中移除当前key对应的锁



## 整体处理逻辑说明

### Get(key) 逻辑

store将handler指为StorageHandle 具体处理指令入口为：/src/storage/txn/scheduler.rs L774 notify 函数

#### notify处理Get(key)请求，实际过程如下：

1. notify收到RawCmd---Get（key）请求，准备锁信息，准备snapshop并通过发送SnapshotFinished至下一步
2. notify收到SnapshotFinished请求，根据snapshot信息处理具体Get,并发送ReadFinished请求
3. noity 收到ReadFinished请求，处理请求内容，并发起请求的回调。

#### 处理RawCmd请求

* notify获取到Get(key)请求
* msg匹配到RawCmd,执行on_receive_new_cmd

* **on_receive_new_cmd**：检查是否执行此命令,目前只对写请求做了限流处理（当前命令为读命令，无论是否繁忙，都往下执行）

* on_receive_new_cmd: 执行schedule_command
* ** schedule_command** ：

	1. 为当前cmd生成唯一编号
	2. 检测并获取当前cmd所需的锁变量条件，读请求不需要锁
	3. 生成执行环境RunningCtx，并将该环境插入到处理map中（该map用cid作为key,用于存取运行过程中共享的信息）
*  执行**lock_and_get_snapshot**:
	1. 获取当前cmd所需的所有锁
	2. 获取snapshot并发送SnapshopFinished通知（异步）

#### Notify处理SnapshotFinished请求
* notify 获取到SnapshotFinished通知,执行**on_snapshot_finished**:
 		1. 如snapshop执行成功，做进一步处理process_by_worker
		2. 如snapshop内容为error,使用error调用请求的回调，结束

* process_by_worker：分发处理读写请求

	1. 从CbContext准备处理环境（根据cid获取到缓存中的RunningCtx信息）
	2. 根据读写，用处理池工具分别处理。这里为读请求，到process_read

* process_read:用一个工作线程处理一个读请求，并在处理完成时，发送ReadFinished通知

	1. 判断请求，并按具体请求分别处理，这里为Get请求。
	2. 使用通知中的snapshop创建SnapshotStore
	3. 定位到当前方法为RawGet, 调用SnapshotStore 的get(key)获取key对应的值
	4. 根据结果：**错误／准确值,发送ReadFinished通知**

* snapshot store get(key)

	1. 基于当前snapshot构建MvccReader 
	2. 获取数据reader.get(key).    

* MvccReader get(key,ts)

	1. 获取当前key的lock，如果当前key的lock存在（即如果当前key－lock的时间发生在当前ts之前)，返回错误
	2. 循环执行以下过程，直到从write  CF 中获取符合条件的key的ts--(commit_ts,write)，返回default CF中对应的真实数据:
      
      * 使用get(key,ts)从write表中获取符合条件commit记录（commit_ts,write）
	  * 若未在write Column Family中找到对应记录，则返回数据不存在。
	  * 若write的类型是PUT /Delete,返回default中对应commit_ts的数据
	  * 如果是Lock或者Rollback状态，ts＝commit_ts-1,继续执行以上过程


#### Notify处理ReadFinished请求

notify收到ReadFinished请求，执行on_read_finished:

1. 从缓存中移除当前cid对应的环境信息
2. 如果存在next command,继续往下处理；否则，带数据执行回调
3. 释放当前请求相关的内存锁 release_lock:释放所有锁，并尝试唤醒所有相关的锁当前的队首请求




