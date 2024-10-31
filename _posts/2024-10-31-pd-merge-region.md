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

