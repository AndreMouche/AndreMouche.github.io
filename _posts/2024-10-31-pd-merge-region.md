---
layout: post
title: "Deep dive into merge-region and common issues"
keywords: ["tidb"]
description: "deep dive into merge-region and common issues"
category: "tidb"
tags: ["tidb","tikv","pd"]
comments: true
---

# Why merge regions

`merge-region`, as the name suggests, refers to merging adjacent small regions of the data range to reduce the total number of regions. The main reasons for merge-region are as follows:

- Maintaining the information of the Region consumes resources: When there are too many small regions, maintaining the metadata information and process the region's heartbeat requires additional resource overhead, which will accumulate to a certain extent and have a performance impact on our cluster.

- Region is the basic unit for most TIKV read and write requests. Therefore, if the region is too small or scattered, it will directly cause the read and write requests to TiKV to become scattered, resulting in requests that cannot be processed in batches. For example, if a 100 M region is divided into 100 1 M regions, then reading the data of this region requires sending 100 requests to KV.

# When do we need to merge regions ?

- When deleted a lot of data in the cluster
    - contiguous space deletion(unsafe-destroy-range): truncate/drop table/partition deletes a large amount of data and generates a large number of empty regions after GC lifetime
    - Scattered space deletion: When there are many mvcc versions for each row due to frequently updates( delete/update/insert), undersized-region/empty-region will be generated after the gc lifetime and compaction is completed, which also needs to be merged

-  Increase the region size since there are too many regions in the cluster and . At this time, many `undersized-regions` will be generated in a short time and merge will begin:

    - adjust the `region-size` to decrease the number of regions in the cluster

```
//set region-split-size(online config)
set config tikv `coprocessor.region-max-size` = '256MiB';
set config tikv `coprocessor.region-max-keys` = '1440000';
// set merge size limit
set config pd `schedule.max-merge-region-size`=20;
set config tikv `schedule.max-merge-region-keys` = '1440000';
```

# Check the progress of `merge-region`
  
When you see the following two metrics in the `pd-> region-health` panel, you need to pay attention:

- `empty-region-count`: the number of regions with size <= 1 M

- `undersized-region-count` : the number of regions that with size < = `schedule.max-merge-region-size` and region keys < = `schedule.max-merge-region-keys` , which is the region that meets the merge condition.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_merge_region/region-healthy.png?raw=true" width="600" />


# Speed up merge region

PD, as the brain of the cluster, controls the health of the entire cluster, and one of its important tasks is ensure the health of the region. When an anomaly is found on a region, for example when we find many empty or `undersized-sized` regions, it will put great pressure on the maintenance of the region metadata when there are many of these regions, thereby affecting the overall performance of the cluster. Therefore, after detecting such anomalies, PD will create corresponding scheduling operators to guide the region in TiKV to complete the self-repair function. For these undersized regions, PD will give the merge-region operator to guide the corresponding region in TiKV to complete the region merge process.

Therefore, to speed up region merging, is to accelerate the following two steps:

- Speed up generating merge-region operator on PD side.

- Speed up consuming region-merge operator on TiKV side

## Speed up generating merge-region operator

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_merge_region/patrol_region.png?raw=true" width="600" />

`Merge-checker` is the last step in PD's region health check . If the region does not encounter any abnormalities in the previous detection process, it will finally go to the merge-checker step. The related configurations are :

- [split-merge-interval](https://docs.pingcap.com/zh/tidb/stable/pd-configuration-file#split-merge-interval): For the newly split region, or when the cluster is just started, you need to wait split-merge-interval before generating a merge-region operator for it.

- [Merge-scheduler-limit](https://docs.pingcap.com/tidb/stable/pd-configuration-file#merge-schedule-limit): 8 by default. Related metrics are pd->operator->Schedulers reach limit

``` 
// update merge-scheduler-limit
tiup ctl:v7.5.2 pd config set merge-schedule-limit 64
Starting component ctl: /home/tidb/.tiup/components/ctl/v7.5.2/ctl pd config set merge-schedule-limit 64
Success!
tidb@172-16-120-219:~$ tiup ctl:v7.5.2 pd config show
Starting component ctl: /home/tidb/.tiup/components/ctl/v7.5.2/ctl pd config show
{
  "replication": {
    ……
  },
  "schedule": {
    ……
    "max-merge-region-keys": 1000000,
    "max-merge-region-size": 100,
    "max-movable-hot-peer-size": 512,
   ……
    "merge-schedule-limit": 64,
    "patrol-region-interval": "10ms",
    ……
  }
}

```

Attention: Store-limit will not affect merge.

### Conditions to create an merge operator (MergeChecker. Check):

When PD performs a `merge-check` on a region, it checks a series of conditions. When the `merge-region` operator fails to create, it also records the details in `pd- > scheduler- > region merge checker`.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_merge_region/region_merge_checker.png?raw=true" width="600" />

With the above metrics, we can quickly find out the reason why the `merge-region` operator has not been generated. We will introduce the indicators in the figure in detail according to the check order of the merge region in this process:

- Paused : merge_checker is disabled
- Recently-start : Within the most recently started split-merge-interval (default 1h)
- Recently-split : Just split within the last split-merge-interval (default 1h) and will not be merged
- skip-uninit-region : region has no leader, usually occurs when pd just started, waiting for the region heartbeat from TiKV
- No-need : The region is relatively large and does not need to be merged. Related configurations: max-merge-region-size && max-merge-region-keys
- special-peer : region has a down or pending replica
- abnormal-replica : region has an exception acccording to the placement-rule, such as missing replicas or replicas not placed according to the replacement-rule
- hot-region : hotspot region not merge
- no-target : Cannot find region to merge with
    - adj-not-exist : cannot find the adjacent region
    - adj-recently-split : the adjacent region just split recently
    - adj-region-hot : the adjacent region is hot
    - adj-disallow-merge : the adjacent regions are not allowed to merge for the following reasons:
        - The two regions have different placement rules
        - One of the regions is labeled `merge_option = deny` .
        - enable-cross-table-merge is false, and the two regions belong to different tables
    - adj-abnormal-peerstore : Both regions to be merged have replicas on the abnormal store (offline or missing)
    - adj-special-peer : the adjacent region has `down-peer` or `pending-peer`
    - adj-abnormal-replica : the adjacent regionan has an exception acccording to the `placement-rule`, such as missing replicas or replicas not placed according to the replacement-rule
    - target-too-large : After merging, the region size will become a super large region, such as larger than `region-max-size` * 4
- split-size-after-merge After merging, the size is greater than `region-max-size`, and it can reach the conditions of split. That is to say, even if it is merged, it will be quickly split again, so the merge will be rejected.
- split-keys-after-merge : similiar as above, only the unit becomes keys. That is, after merging, it will soon reach the split condition and will be split again.
- Larger-source : The size or number of keys in the source region is greater than the merged region, just mark it, and this operator will still be generated.

### key configrations in PD related to merge-region

- [max-merge-region-size](https://docs.pingcap.com/tidb/v7.5/pd-configuration-file#max-merge-region-size)(20M by default):Controls the size limit of Region Merge. When the Region size is greater than the specified value, PD does not merge the Region with the adjacent Regions..
- [max-merge-region-keys](https://docs.pingcap.com/tidb/v7.5/pd-configuration-file#max-merge-region-keys)(20 million by default):Specifies the upper limit of the Region Merge key. When the Region key is greater than the specified value, the PD does not merge the Region with its adjacent Regions.
- [split-merge-interval](https://docs.pingcap.com/tidb/v7.5/pd-configuration-file#split-merge-interval)(1h by default): Controls the time interval between the split and merge operations on the same Region. That means a newly split Region will not be merged for a while.
- [merge-schedule-limit](https://docs.pingcap.com/tidb/v7.5/pd-configuration-file#merge-schedule-limit)(8 by default):The number of the Region Merge scheduling tasks performed at the same time. Set this parameter to 0 to disable Region Merge.
- [enable-cross-table-merge](https://docs.pingcap.com/tidb/v7.5/pd-configuration-file#enable-cross-table-merge)(true by default):Determines whether to enable the merging of cross-table Regions

## Speed up merge-region operator's consumption 

Talking about the consumption speed of `merge-region`, first we need to know is how a `region-merge` operator is done.

Region itself is a logical concept. As the smallest logical unit of TiKV storage, the data replica behind it is physically stored on TiKV. Therefore, when doing region merge, we first adjust the topology structure of the data replicas and roles of the two regions to be merged to be completely consistent, and then we can safely and quickly merge them logically.

Below, we will take region-2 [B, C) and region-3 [C, D) merged into region-2 [B, D) as an example to briefly show the process of the region-merge operator.

In initialization state:

- The data range of Region-2 is [B, C), and the three replicas for this region-2 are on store-1, store-2, and store-3, with the leader on store-2
- The data range of Region-3 is [C, D), with three replicas located in store-1, store-2, and store-4, with the leader located in store-4.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_merge_region/merge-region-1.png?raw=true" width="600" />

To merge region-2 and region-3, first we need to move the data replicas of the two regions to the same store. Assuming at this stage, we will move the replica of region-2 from store-3 to store-4.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_merge_region/merge-region-2.png?raw=true" width="600" />


Now, the two regions each have 1 replica on store-1, store-2, and store-4. However, to merge them, we also need the replica's role distribution topology of these two regions to be completely consistent. Therefore, here we transfer the leader of region-2 to the node store-4 where the leader of region-3 is located.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_merge_region/merge-region-3.png?raw=true" width="600" />


Now that the replica role topologies of region-2 and region-3 are completely consistent, we can safely complete the logical merge.

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/tidb_merge_region/merge-region-4.png?raw=true" width="600" />


From the above execution principle of `merge-region`, we can easily understand that for `merge-region`, the corresponding region's data replica needs to be relocated to the same store during the merge process. Therefore, the configuration related to TiKV replica relocation will affect the speed of `merge-regio`n execution. The most resource-consuming and also the easiest to become a bottleneck is the data relocation that named `add-learner`. You can check more details in  [Deep dive into data relocation between tikv](https://andremouche.github.io/tidb/tidb-move-region-between-stores.html) . And the key configurations in this step are:

- [snap-generator-pool-size](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#snap-generator-pool-size-new-in-v540) (default 2), used by `snap-generator`
-  [snap-io-max-bytes-per-sec](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#snap-io-max-bytes-per-sec) (100MB by default),used by `snap-generator`
- [concurrent-send-snap-limit](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#concurrent-send-snap-limit)(32 by default) used by `snap_handler`.`snap_sender`
- [concurrent-recv-snap-limit](https://docs.pingcap.com/tidb/v7.5/tikv-configuration-file#concurrent-recv-snap-limit) (32 by default) used by `snap_handler`.`snap_sender` 


## Check the progress of a specific region-merge through PD logs

Log example:

You can refer to this log when a specific merge-region operator gets stuck

```
// create a merge-region operator: merge: region 512607784 to 513499180 
/** steps:
add learner peer 612422577 on store 16910359, 
promote learner peer 612422577 on store 16910359 to voter, 
remove peer on store 16910361, 
add learner peer 612422578 on store 266988190, 
promote learner peer 612422578 on store 266988190 to voter, 
remove peer on store 266988760, 
add learner peer 612422579 on store 536155773, 
promote learner peer 612422579 on store 536155773 to voter, 
transfer leader from store 536156435 to store 8, 
remove peer on store 536156435, 
add learner peer 612422580 on store 142072119, 
promote learner peer 612422580 on store 142072119 to voter, 
remove peer on store 536156437, 
add learner peer 612422576 on store 266988192, 
promote learner peer 612422576 on store 266988192 to voter, 
transfer leader from store 8 to store 16910359, 
remove peer on store 8, merge region 512607784 into region 513499180])\
**/
// The merge operator is always created in pairs on two regions that need to be merged.
[2024/07/22 20:02:58.803 +08:00] [INFO] [operator_controller.go:424] ["add operator"] [region-id=512607784] [operator="\"merge-region {merge: region 512607784 to 513499180} (kind:leader,region,merge, region:512607784(244793,40956), createAt:2024-07-22 20:02:58.803075332 +0800 CST m=+27423622.512140371, startAt:0001-01-01 00:00:00 +0000 UTC, currentStep:0, steps:[add learner peer 612422577 on store 16910359, promote learner peer 612422577 on store 16910359 to voter, remove peer on store 16910361, add learner peer 612422578 on store 266988190, promote learner peer 612422578 on store 266988190 to voter, remove peer on store 266988760, add learner peer 612422579 on store 536155773, promote learner peer 612422579 on store 536155773 to voter, transfer leader from store 536156435 to store 8, remove peer on store 536156435, add learner peer 612422580 on store 142072119, promote learner peer 612422580 on store 142072119 to voter, remove peer on store 536156437, add learner peer 612422576 on store 266988192, promote learner peer 612422576 on store 266988192 to voter, transfer leader from store 8 to store 16910359, remove peer on store 8, merge region 512607784 into region 513499180])\""] ["additional info"=]
[2024/07/22 20:02:58.803 +08:00] [INFO] [operator_controller.go:424] ["add operator"] [region-id=513499180] [operator="\"merge-region {merge: region 512607784 to 513499180} (kind:leader,region,merge, region:513499180(244859,40965), createAt:2024-07-22 20:02:58.803076494 +0800 CST m=+27423622.512141533, startAt:0001-01-01 00:00:00 +0000 UTC, currentStep:0, steps:[merge region 512607784 into region 513499180])\""] ["additional info"=]
[2024/07/22 20:02:58.804 +08:00] [INFO] [operator_controller.go:620] ["send schedule command"] [region-id=513499180] [step="merge region 512607784 into region 513499180"] [source=create]
[2024/07/22 20:03:04.264 +08:00] [INFO] [operator_controller.go:620] ["send schedule command"] [region-id=513499180] [step="merge region 512607784 into region 513499180"] [source="active push"]
[2024/07/22 20:03:09.267 +08:00] [INFO] [operator_controller.go:620] ["send schedule command"] [region-id=513499180] [step="merge region 512607784 into region 513499180"] [source="active push"]
[2024/07/22 20:03:14.764 +08:00] [INFO] [operator_controller.go:620] ["send schedule command"] [region-id=513499180] [step="merge region 512607784 into region 513499180"] [source="active push"]
[2024/07/22 20:03:16.293 +08:00] [INFO] [operator_controller.go:620] ["send schedule command"] [region-id=512607784] [step="merge region 512607784 into region 513499180"] [source=heartbeat]
[2024/07/22 20:03:16.388 +08:00] [INFO] [cluster.go:545] ["region Version changed"] [region-id=513499180] [detail="StartKey Changed:{7480000000000827FF7C5F7280000000C6FF7CFD9E0000000000FA} -> {7480000000000827FF7C5F7280000000C5FF5284990000000000FA}, EndKey:{7480000000000827FF7C5F7280000000C7FF8376770000000000FA}"] [old-version=244859] [new-version=244860]
[2024/07/22 20:03:16.407 +08:00] [INFO] [operator_controller.go:537] ["operator finish"] [region-id=513499180] [takes=17.603823476s] [operator="\"merge-region {merge: region 512607784 to 513499180} (kind:leader,region,merge, region:513499180(244859,40965), createAt:2024-07-22 20:02:58.803076494 +0800 CST m=+27423622.512141533, startAt:2024-07-22 20:02:58.803993629 +0800 CST m=+27423622.513058670, currentStep:1, steps:[merge region 512607784 into region 513499180]) finished\""] ["additional info"=]

// in expection, since the region 512607784 has been merged into 513499180, so the operator on it does not need to be processed anymore, this operator will be cleaned up.
[2024/07/22 20:03:19.768 +08:00] [WARN] [operator_controller.go:211] ["remove operator because region disappeared"] [region-id=512607784] [operator="merge-region {merge: region 512607784 to 513499180} (kind:leader,region,merge, region:512607784(244793,40956), createAt:2024-07-22 20:02:58.803075332 +0800 CST m=+27423622.512140371, startAt:2024-07-22 20:02:58.803798029 +0800 CST m=+27423622.512863076, currentStep:17, steps:[add learner peer 612422577 on store 16910359, promote learner peer 612422577 on store 16910359 to voter, remove peer on store 16910361, add learner peer 612422578 on store 266988190, promote learner peer 612422578 on store 266988190 to voter, remove peer on store 266988760, add learner peer 612422579 on store 536155773, promote learner peer 612422579 on store 536155773 to voter, transfer leader from store 536156435 to store 8, remove peer on store 536156435, add learner peer 612422580 on store 142072119, promote learner peer 612422580 on store 142072119 to voter, remove peer on store 536156437, add learner peer 612422576 on store 266988192, promote learner peer 612422576 on store 266988192 to voter, transfer leader from store 8 to store 16910359, remove peer on store 8, merge region 512607784 into region 513499180])"]

[2024/07/22 20:03:19.768 +08:00] [INFO] [operator_controller.go:572] ["operator canceled"] [region-id=512607784] [takes=20.964336678s] [operator="\"merge-region {merge: region 512607784 to 513499180} (kind:leader,region,merge, region:512607784(244793,40956), createAt:2024-07-22 20:02:58.803075332 +0800 CST m=+27423622.512140371, startAt:2024-07-22 20:02:58.803798029 +0800 CST m=+27423622.512863076, currentStep:17, steps:[add learner peer 612422577 on store 16910359, promote learner peer 612422577 on store 16910359 to voter, remove peer on store 16910361, add learner peer 612422578 on store 266988190, promote learner peer 612422578 on store 266988190 to voter, remove peer on store 266988760, add learner peer 612422579 on store 536155773, promote learner peer 612422579 on store 536155773 to voter, transfer leader from store 536156435 to store 8, remove peer on store 536156435, add learner peer 612422580 on store 142072119, promote learner peer 612422580 on store 142072119 to voter, remove peer on store 536156437, add learner peer 612422576 on store 266988192, promote learner peer 612422576 on store 266988192 to voter, transfer leader from store 8 to store 16910359, remove peer on store 8, merge region 512607784 into region 513499180])\""]
```

# Recent optimization(PRs) for `merge-region` on TiKV side

- Significant improvement in replica migration performance for small regions such as empty regiontikv [#17408](https://github.com/tikv/tikv/pull/17408)

  - For small region data transfer, for regions smaller than `snap_min_ingest_size` (default 2MB), the original ingest method will be changed to write directly to rocksdb. For merge-region situations with empty regions (< 1M), there will be significant performance improvement

- The problem of region-worker getting stuck due to `delete-range` and `generate-snapshot` tasks during data transfer has also been effectively alleviated in tikv [#12587](https://github.com/tikv/tikv/issues/12587) .


# Common incident and workaround

## Empty regions cannot be merged: Different Placement-rules cause adjacent regions to not be merged (as expected).

- Phenomenon: When the user has enabled the [`enable-cross-table-merge`](https://github.com/tikv/tikv/issues/12587) , multiple placement rules are set, such as table-1, table-2, and table-3 have different placement-rules.
  - Table 1 is required to be placed in Beijing and Shanghai
  - Table 2 requires to be placed in Hangzhou, Shandong
  - Table 3 requires placement in Beijing and Shanghai

Among them, the table-id of table 1\ table 2\ table3 is continuous, that is, the data area is continuous, that is, the data range of the region where they are located is continuous

At this time, if table-2 is dropped and an empty region is generated, this empty region cannot be merged with the region of table-1 or table-3, and this time the empty region cannot be merged.

- Workaround: Set the placement-rule of table2 to be the same as table-1 or table-3.

## Merge region related memory leaks

- Phenomenon: A large number of merge-region operations may cause memory leaks in statistical information, resulting in a continuous increase in PD memory
  - The merged region's delayed heartbeat may cause the information of the current region to remain in the PD memory forever [pd#8710](https://github.com/tikv/pd/issues/8710) [pd#8700](https://github.com/tikv/pd/issues/8700)
  - The merged region cannot clear the cache left in the hot-region because it will no longer send heartbeats [pd#8698](https://github.com/tikv/pd/issues/8698)

- workaround：
  - Periodic restart PD
  - Upgrade to a fixed version