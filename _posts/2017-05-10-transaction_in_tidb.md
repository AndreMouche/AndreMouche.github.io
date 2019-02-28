---
layout: post
title: "Transaction in TiDB"
keywords: ["tidb"]
description: "Transaction in TiDB"
category: "tidb"
tags: ["tidb","transaction","tikv"]
comments: true
---


# Transaction in TiDB

`TiDB` 的事务模型参考了 [Percolator](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf) 论文，Percolator 事务模型详见上一篇文章-[Google Percolator 的事务模型](http://andremouche.github.io/transaction/percolator.html)

本文将详细介绍事务在 `tidb` 中实现，主要内容包括

*  基本概念
* `tidb` 中一致性事务的实现
* `tikv` 中事务相关的接口逻辑
* `tidb` 事务如何做到 `ACID`

## TIKV


<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/arch.jpeg?raw=true" width="600" />

`tidb` 为 `tikv` 的客户端，对外提供兼容 `mysql` 协议的分布式关系型数据库服务。

pd 提供两大功能

* 提供包括物理时间的全局唯一递增时间戳 tso
* 管理 raft-kv 集群

`tikv` 对外提供分布式 `kv` 存储引擎，同时实现了 `mvcc` 相关的接口－－方便客户端实现 `ACID` 事务。


### Columns in TIKV

 `tikv` 底层用  `raft+rocksdb` 组成的 `raft-kv` 作为存储引擎，具体落到 `rocksdb` 上的 `column` 有四个，除了一个用于维护 `raft` 集群的元数据外，其它 3 个皆为了保证事务的 `mvcc`, 分别为 `lock`, `write`, `default`，详情如下：
 
#### Lock

事务产生的锁，未提交的事务会写本项，会包含`primary lock`的位置。其映射关系为

```
${key}=>${start_ts,primary_key,..etc}
```

#### Write

已提交的数据信息，存储数据所对应的时间戳。其映射关系为

```
${key}_${commit_ts}=>${start_ts}
```


#### Default(data)

具体存储数据集，映射关系为

```
${key}_${start_ts} => ${value}
```

## Primary Key

`TiDB` 对于每个事务，会从涉及到改动的所有 `Key` 中选中一个作为当前事务的 `Primary Key`，事务的状态将保存在这个 `Key` 上。 在最终提交时，以 `Primary` 提交是否成功作为整个事务是否执行成功的标识，从而保证了分布式事务的原子性。

有了 `Primary key` 后，简单地说事务两阶段提交过程如下：

1. 从当前事务涉及改动的 keys 选中一个作为 `primary key`, 剩余的则为 `secondary keys`
2. 并行 `prewrite` 所有 `keys`。 这个过程中，所有 `key` 会在系统中留下一个指向 `primary key` 的锁。
3. 第二阶段提交时，首先 `commit` primary key ,若此步成功，则说明当前事务提交成功。
4. 异步并行 `commit secondary keys` 

一个读取过程如下：

1. 读取 `key` 时，若发现没有冲突的锁，则返回对应值，结束。
2. 若发现了锁，且当前锁对应的 `key` 为 `primary`： 若锁尚未超时，等待。若锁已超时，`Rollback` 它并获取上一版本信息返回，结束。
3. 若发现了锁，且当前锁对应的 `key` 为 `secondary`, 则根据其锁里指定的 `primary` 找到 `primary`所在信息，根据 `primary` 的状态决定当前事务是否提交成功，返回对应具体值。

## TIDB 事务处理流程

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/2pc.png?raw=true" width="600" />

注意：所有涉及重新获取 tso 重启事务的两阶段提交的地方，会先检查当前事务是否可以满足重试条件：只有单条语句组成的事务才可以重新获取 tso 作为 start_ts。

1. `client` 向 `tidb` 发起开启事务 `begin`
2. `tidb` 向 `pd` 获取 `tso` 作为当前事务的 `start_ts`
3. `client` 向 `tidb` 执行以下请求：
	* 读操作，从 `tikv` 读取版本 `start_ts` 对应具体数据.
	* 写操作，写入 `memory` 中。
4. `client` 向 `tidb` 发起 `commit` 提交事务请求
5. `tidb` 开始两阶段提交。
6. `tidb` 按照 `region` 对需要写的数据进行分组。
7. `tidb` 开始 `prewrite` 操作：向所有涉及改动的 `region` 并发执行 `prewrite` 请求。若其中某个`prewrite` 失败，根据错误类型决定处理方式：
	* `KeyIsLock`：尝试 `Resolve Lock` 后，若成功，则重试当前 region 的 `prewrite`[步骤7]。否则，重新获取 `tso` 作为 `start_ts ` 启动 2pc 提交（步骤5）。
	* `WriteConfict` 有其它事务在写当前 `key`, 重新获取 `tso` 作为 `start_ts ` 启动 2pc 提交（步骤5）。
	* 其它错误，向 `client` 返回失败。
8. `tidb` 向 `pd` 获取 tso 作为当前事务的 `commit_ts`。
9. `tidb` 开始 `commit`:`tidb` 向 `primary` 所在 `region` 发起 `commit`。 若 `commit primary` 失败，则先执行 `rollback keys`,然后根据错误判断是否重试:
    * `LockNotExist` 重新获取 `tso` 作为 `start_ts ` 启动 2pc 提交（步骤5）。
    * 其它错误，向 `client` 返回失败。
10. `tidb` 向 `tikv` 异步并发向剩余 `region` 发起 `commit`。
11. `tidb` 向 `client` 返回事务提交成功信息。


## TiKV 事务处理细节

### Prewrite

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/prewrite.png?raw=true" width="600" />

Prewrite 是事务两阶段提交的第一步，其从`pd`获取代表当前物理时间的全局唯一时间戳作为当前事务的 `start_ts`，尝试对所有被写的元素加锁(为应对客户端故障，`tidb` 为所有需要写的 key 选出一个作为 `primary`,其余的作为`secondary`)，将实际数据存入 `rocksdb`。其中每个key的处理过程如下，中间出现失败，则整个`prewrite`失败：

1. 检查 `write-write` 冲突：从 `rocksdb` 的`write` 列中获取当前 `key` 的最新数据，若其 `commit_ts` 大于等于`start_ts`,说明存在更新版本的已提交事务，向 `tidb` 返回 `WriteConflict` 错误，结束。
2. 检查 `key` 是否已被锁上，如果 `key` 的锁已存在，收集 `KeyIsLock` 的错误，处理下一个 `key`
5. 往内存中的 `lock` 列写入 `lock(start_ts,key)` 为当前 key 加锁,若当前 key 被选为 `primary`, 则标记为 `primary`,若为 `secondary`,则标明指向 `primary` 的信息。
6. 若当前 `value` 较小，则与 `lock` 存在一起，否则，内存中存入 `default(start_ts,key,value)`。

处理完所有数据后，若存在 `KeyIsLock` 错误，则向 `tidb` 返回所有 `KeyIsLocked` 信息。
否则，提交数据到 `raft-kv` 持久化，当前 `prewrite` 成功。


### Commit 

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/commit.png?raw=true" width="600" />

1. `tidb` 向 `tikv` 发起第二阶段提交 `commit(keys,commit_ts,start_ts)`
2. 对于每个 key 依次检查其是否合法，并在内存中更新（步骤3-4）
3. 检查 key 的锁，若锁存在且为当前 start_ts 对应的锁，则在内存中添加 write(key,commit_ts,start_ts),删除 lock(key,start_ts)，继续执行下一个 key(跳至步骤3)。否则，执行步骤 4
4. 获取当前 key 的 `start_ts` 对应的数据 `write(key,start_ts,commit_ts)`, 若存在，说明已被提交过，继续执行下一个 key(跳至步骤4)。否则，返回未找到锁错误到 `tidb`，结束。
5. 到底层 raft-kv 中更新 2-4 步骤产生的所有数据－－这边保证了原子性。
6. `tikv` 向 `tidb` 返回 `commit` 成功。

### Rollback

当事务在两阶段提交过程中失败时， `tidb` 会向当前事务涉及到的所有 `tikv` 发起回滚操作。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/rollback.png?raw=true" width="600" />
    
 1. ｀tidb｀ 向 `tikv` 发起 `rollback(keys,start_ts)`, 回滚当前 `region` 中 `start_ts` 所在的 key 列表
 2.  对于每个 `key`, `tikv` 依次检查其合法性，并进行回滚(依次对每个 key 执行 3-4)
 3.  检查当前 `key` 的锁，情况如下：	 
 	  * 若当前 (start_ts,key) 所对应的锁存在，则在内存中删除该锁,继续回滚下一个 `key`(跳至步骤2）
 	  * 若当前事务所对应的锁不存在，则进入步骤 4 检查提交情况
 4.  `tikv` 从 `raft-kv` 中获取到当前 (key,start_ts) 对应的提交纪录：
     * 若 commit(key,commit_ts,start_ts) 存在，且状态为PUT／DELETE， 则返回 `tidb` 告知事务已被提交，回滚失败，结束。
     * 若 commit(key,commit_ts,start_ts) 存在，且状态为 `Rollback`, 说明当前 `key` 已被 `rollback` 过，继续回滚下一个 `key`(跳至步骤2)
     * 若 (key,start_ts) 对应的提交纪录不存在，说明当前 `key` 尚未被 `prewrite` 过，为预防 `prewrite` 在之后过来，在这里留下 `(key,start_ts,rollback)`, 继续回滚下一个 `key`(跳至步骤2)
 
 5. 将步骤 2-4 中更新的内容持久化到 `raft-kv`
 6. `tikv` 向 `tidb` 返回回滚成功。

### Resolve Lock

`tidb` 在执行 `prewrite`, `get` 过程中，若遇到锁，在锁超时的情况下，会向 `tikv` 发起清锁操作。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/resolve.png?raw=true" width="600" />

1. `tidb` 向 `tikv` 发起 `Resolve(start_ts,commit_ts)`
2. `tikv` 依次找出所有 `lock.ts==start_ts` 的锁并执行对应的清锁操作（循环执行步骤3-5）。
3. `tikv` 向 `raft-kv` 获取下一堆`lock.ts==start_ts` 的锁。
4. 若没有剩余锁，退出循环，跳至步骤6。
5. 对本批次获取到的锁，根据情况进行清锁操作
		* 若 commit_ts 为空，则执行回滚
		* 若 commit_ts 有值，则执行提交。
6. 将2-5 产生的更新持久化到 `raft-kv`
7. `tikv` 向 `tidb` 返回清锁成功。


 
 ### Get
 
  <img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/get.png?raw=true" width="600" />
  
 1. `tidb` 向 `tikv` 发起 `get(key,start_ts)` 操作
 2.  `tikv` 检查当前 `key` 的锁状态：若锁存在且 `lock.ts<start_ts`, 即 `start_ts` 这一刻该值是被锁住的，则返回 `tidb` 被锁住错误，结束。
 3. 尝试获取 `start_ts` 之前的最近的有效提交，初始化 版本号`version` 为 `start_ts－1`
 4. 从 `write` 中获取 commit_ts<=version 的最大 commit_ts 对应纪录：
 		* 若 `write_type=PUT`,即有效提交，返回 `tidb` 当前版本对应值。
 		* 若 `write 不存在或者 write_type=DEL`, 即没有出现过该值，或该值最近已被删除，返回`tidb`空
 		* 若 `write_type=rollback,commit_ts`，则 version=commit_ts-1, 继续查找下一个最近版本（跳至步骤3）
 		


### GC

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/gc.png?raw=true" width="600" />

1. `tidb` 向 `tikv` 发起 `GC`操作，要求清理 `safe-point` 版本之前的所有无意义版本
2. `tikv` 分批进行 `GC`
3. `tikv` 向 `raft-kv` 从 `write`  列中获取一批提交纪录，若上一批已处理过，则从上一批最后一个开始扫。若未获取到数据，则本次 `GC` 完成，返回 `tidb` 成功。
4. `tikv` 对本批次的所有 `key` 挨个进行GC:
   *  清理所有小于 `safe-point` 的 `Rollback` 和 `Lock`
   *  若小于 `safe-point` 的除了`Rollback` 和 `Lock`外第一个为 `delete`, 清理所有`safe-point` 之前的数据。
   *  若小于 `safe-point` 的除了`Rollback` 和 `Lock`外第一个为 `PUT`, 保留该提交，清理所有该提交之前的数据。
5. `tikv` 向 `raft-kv` 更新步骤 4 的改动。进入下一批清理 （步骤3）.



## ACID in TIDB

### 原子性
	两阶段提交时，通过 `primary key` 保证原子性。 `primary key` commit 成功与否决定事务成功与否。
	
### 一致性

### 隔离性

通过两阶段提交，保证隔离级别为 `RR`

### 持久性

tikv 保证持久化。

## Why 2 PC?

###  1 PC 存在的问题

1 pc 无法保证隔离性为 `RR`.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/1pc.png?raw=true" width="600" />

假设初始状态下，`(A,t0)=>v1`, 以下操作有序发生时：

1. txn1: 向 `pd` 获取 `tso1` 作为当前事务的 `start_ts`。
2. txn2: 向 `pd` 获取 `tso2` 作为当前事务的 `start_ts`, 此时 `tso2>tso1`
3. txn2: 向 `pd` 获取 `A`, 此时获取到 `A` 在 `tikv` 上的小于等于 `tso2` 的最新版本为 `v1`.
4. txn1: 向 `pd` 更新 `A`为 `v2`, 此时`tikv` 上有记录：（A,tso1）=>v2
5. txn2: 向 `pd` 获取 `A`, 此时获取到 `A` 在 `tikv` 上的小于等于 `tso2` 的最新版本为 `v2`.

### 2 PC 保证隔离级别

两阶段提交简介：
第一阶段 prewrite:
   1. 获取一个包含当前物理时间的、全局唯一递增的时间戳 t1 作为当前事务的 start_ts
   2. lock(key,start_ts), save data(key,start_ts,data)
第二阶段 commit 数据：
   1. 获取一个包含当前物理时间的、全局唯一递增的时间戳 t2 作为当前事务的 commit_ts
   2. commit(key,commit_ts, start_ts)

事务读取数据步骤如下：
   1. 获取一个包含当前物理时间的、全局唯一递增的时间戳 t1 作为当前事务的 start_ts.
   2. 检查当前查询 key 的锁，若锁存在，且 lock.ts<t1, 说明这一刻 key 正在被写入，需要等待写入事务完成再读取。
   3. 到这一步时，说明 要么 锁不存在，要么 lock.ts > t1, 这两种情况都能说明， 下一个该 key 的 commit_ts 一定会大于当前的 t1, 所以可以直接读取当前小于 t1 的最大 commit_ts 对应的数据。

   
综上，两阶段提交可以保证事务的隔离级别为 RR，示例如下：

假设初始状态下，`(A,t0)=>v1`, 现有事务1－txn1, 事务2-txn2。其中 txn-1 将会修改 A 的值为 v2。假设 txn－1 的 start_ts=s1, commit_ts=c1, 事务 txn-2 的 start_ts=s2,那么有：

* 若 s1>s2, 那么 txn2 读取的是 s2 之前的数据，无论如何都不会读取到 txn-1 的更新的数据。

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/img/txn_in_tidb/no_lock.jpg?raw=true" width="600" />

* 若 s1 < s2, 则有以下两种情况。
	1. 若 s2 读取 A 时， 若未发现lock(s1)， 如上图中，txn-2 第一次获取数据时，此时 txn-1 尚未提交，由于 commit_ts 只有在 prewrite 完成后才能获取，所以可以保证 c1>s2, 也就是说，s2 读取不到 txn-1 的数据。
	2. 若 s2 读取 A 时， 若发现了 lock(s1), 如上图中，txn-2 第二次获取数据时的场景， 此时无法确定 c1 与 s2 的大小，所以此时 txn-2 会等待 txn-1 commit, 当 txn-1 commit 结束后，txn-2 才会根据 txn-1 的 commit_ts 确定是否获取 txn-1 的更新数据。也就是说，发现有锁时，txn-2 一定会等待直到确定与 txn-1 的 commit_ts 的大小才会决定获取哪份数据。

综上可见， 两阶段提交能很好的保证事务发生的时序，从而保证了事务的隔离级别为 可重复读（RR）



