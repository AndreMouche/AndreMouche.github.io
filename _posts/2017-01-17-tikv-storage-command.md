---
layout: post
title: "[Tidb]Tikv-storage源码阅读-command"
keywords: ["tidb"]
description: "[Tikv] tikv-storage源码阅读"
category: "tidb"
tags: ["tidb"]
comments: true
---

## Tikv-storage 层源码阅读--Command

项目地址: [github/pingcap/tikv](https://github.com/pingcap/tikv)

## RawCmd

### Get

数据结构：

```
	Get {
        ctx: Context,
        key: Key,
        start_ts: u64,
    }
```

根据start_ts获取key的信息

#### 执行过程

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_get_key_ts.png?raw=true" alt="tikv_get_key_ts.png" title="tikv_get_key_ts.png" width="600" />

1. 获取当前key的lock，如果当前key的lock存在（即如果当前key－lock的时间发生在当前ts之前)，返回错误
2. 循环执行以下过程，直到从write  CF 中获取符合条件的key的ts--(commit_ts,write)，返回default CF中对应的真实数据:
	1. 使用get(key,ts)从write表中获取符合条件commit记录（commit_ts,write）
	2. 若未在write Column Family中找到对应记录，则返回数据不存在。
	3. 若write的类型是PUT /Delete,返回default中对应commit_ts的数据
	4. 如果是Lock或者Rollback状态，ts＝commit_ts-1,继续执行以上过程




### BatchGet

```
    BatchGet {
        ctx: Context,
        keys: Vec<Key>,
        start_ts: u64,
    }
```

批量获取指定keys的信息

#### 执行过程

对每个key,挨个执行get(key)


### Scan

```    
    Scan {
        ctx: Context,
        start_key: Key,
        limit: usize,
        start_ts: u64,
        options: Options,
    }
 ```
 
 * 获取版本号为start_ts的，从start_key开始的limit个key的信息
 * Options里可指定是否只获取key的scan选项

#### 执行过程

TODO 
 
### Prewrite

``` 
    Prewrite {
        ctx: Context,
        mutations: Vec<Mutation>,
        primary: Vec<u8>,
        start_ts: u64,
        options: Options,
    }
``` 

* 事务两阶段提交

#### 执行过程

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_prewrite.png?raw=true" alt="tikv_prewrite.png" title="tikv_prewrite.png" width="600" />

1. 对于prewrite的每个key,执行以下步骤，一步失败则prewrite失败
1. 从lock中获取key的锁，如果锁存在且不是当前prewrite的锁，返回当前事务冲突
2. 在lock中为当前事务的key设置lock,如果当前value较短，则将该值存于lock中，返回
3. 若当前value较长，则将当前key_ts＝》value存入default Column Family中


    
### Commit    
    
 ``` 
    Commit {
        ctx: Context,
        keys: Vec<Key>,
        lock_ts: u64,
        commit_ts: u64,
    }
```

#### 执行过程

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb/tikv_commit.png?raw=true" alt="tikv_commit.png" title="tikv_commit.png" width="600" />

1. 对于commit的每个key,执行以下步骤，一步失败则commit失败
2. 检查当前事务的lock是否存在，若不存在，则返回错误
3. 将当前key,commit_ts=>start_ts存入write 所在CF
4. 从lock CF 中移除当前key对应的锁

    
    
### Cleanup    
    ``` 
    Cleanup {
        ctx: Context,
        key: Key,
        start_ts: u64,
    }
    ``` 

### Rollback
    
    ``` 
    Rollback {
        ctx: Context,
        keys: Vec<Key>,
        start_ts: u64,
    }
    ```
    
### ScanLock

    ``` 
    ScanLock { ctx: Context, max_ts: u64 },
    ``` 
    
### ResolveLock

    ``` 
    ResolveLock {
        ctx: Context,
        start_ts: u64,
        commit_ts: Option<u64>,
        scan_key: Option<Key>,
        keys: Vec<Key>,
    }
    ``` 
    
### Gc
   
    ``` 
    Gc {
        ctx: Context,
        safe_point: u64,
        scan_key: Option<Key>,
        keys: Vec<Key>,
    }
    ```
    
### RawGet    
``` 
    RawGet { ctx: Context, key: Key },
```





