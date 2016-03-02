---
layout: post
title: "[译]weed-fs api(Master Server)"
keywords: ["weed-fs", "storage"]
description: "api of weed-fs"
category: "weed-fs"
tags: ["weed-fs","storage"]

---

## API OF WEED-FS(Master Server)
原文链接[master-server](http://weed-fs.readthedocs.org/en/latest/api.html#master-server)

所有接口均可通过使用*&pretty=y*查看格式化的json串结果

### Assign a file key(分配文件key)

* 基本用法

```
# Basic Usage:
curl http://localhost:9333/dir/assign
{"count":1,"fid":"3,01637037d6","url":"127.0.0.1:8080",
 "publicUrl":"localhost:8080"}
```

* 指定复制类型

```
# To assign with a specific replication type:
curl "http://localhost:9333/dir/assign?replication=001"
```

* 指定申请的文件个数

```
# To specify how many file ids to reserve
curl "http://localhost:9333/dir/assign?count=5"
```

* 指定数据中心
 
```
# To assign a specific data center
curl "http://localhost:9333/dir/assign?dataCenter=dc1"
```

###Lookup volume(查找volumne信息)

该接口用于确定指定volume是否已被移除

```
curl "http://localhost:9333/dir/lookup?volumeId=3&pretty=y"
{
  "locations": [
    {
      "publicUrl": "localhost:8080",
      "url": "localhost:8080"
    }
  ]
}
```

其它用法

* 根据文件id查找

```
# You can actually use the file id to lookup
curl "http://localhost:9333/dir/lookup?volumeId=3,01637037d6"
```

* 根据collection查找会快些

```
# If you know the collection, specify it since it will be a little faster
curl "http://localhost:9333/dir/lookup?volumeId=3&collection=turbo"
```
### Force garbage collection(强制垃圾收集)
当系统中存在大量删除操作时，已删除文件所在的空间不会同步被回收。
系统使用一个后台作业去检测volume空间的使用情况，若可用空间大于系统阀值threshold(默认为0.3),那么：

1. vacuum作业将当前volume设为只读状态
2. vacuum创建一个新的volume,将当前volume中可用文件拷贝入其中
3. 切换到新volume.

在没有足够耐心等待这个过程、或者想做一些测试时，可通过以下方式改变这个动作(TODO：第一个案例不太理解，是强制执行垃圾收集的意思吗？)

```
curl "http://localhost:9333/vol/vacuum"
curl "http://localhost:9333/vol/vacuum?garbageThreshold=0.4"
```

上面这个garbageThreshold是可选参数，该参数不会改变默认的threshold.用户依旧可以指定一个不同的garbageThreshold启动一个新的volume master

###Pre-Allocate Volumes(预分配Volume)
一个Volume一次只能处理一个写操作。如果需要提升并发，用户可以批量预分配volume

生成4个空volume

```
curl "http://localhost:9333/vol/grow?replication=000&count=4"
{"count":4}
```

* 指定collection

```
# specify a collection
curl "http://localhost:9333/vol/grow?collection=turbo&count=4"
```

* 指定数据中心

```
# specify data center
curl "http://localhost:9333/vol/grow?dataCenter=dc1&count=4"
```

###Check System Status(检测系统状态)

```
curl "http://10.0.2.15:9333/cluster/status?pretty=y"
{
  "IsLeader": true,
  "Leader": "10.0.2.15:9333",
  "Peers": [
    "10.0.2.15:9334",
    "10.0.2.15:9335"
  ]
}
curl "http://localhost:9333/dir/status?pretty=y"
{
  "Topology": {
    "DataCenters": [
      {
        "Free": 3,
        "Id": "dc1",
        "Max": 7,
        "Racks": [
          {
            "DataNodes": [
              {
                "Free": 3,
                "Max": 7,
                "PublicUrl": "localhost:8080",
                "Url": "localhost:8080",
                "Volumes": 4
              }
            ],
            "Free": 3,
            "Id": "DefaultRack",
            "Max": 7
          }
        ]
      },
      {
        "Free": 21,
        "Id": "dc3",
        "Max": 21,
        "Racks": [
          {
            "DataNodes": [
              {
                "Free": 7,
                "Max": 7,
                "PublicUrl": "localhost:8081",
                "Url": "localhost:8081",
                "Volumes": 0
              }
            ],
            "Free": 7,
            "Id": "rack1",
            "Max": 7
          },
          {
            "DataNodes": [
              {
                "Free": 7,
                "Max": 7,
                "PublicUrl": "localhost:8082",
                "Url": "localhost:8082",
                "Volumes": 0
              },
              {
                "Free": 7,
                "Max": 7,
                "PublicUrl": "localhost:8083",
                "Url": "localhost:8083",
                "Volumes": 0
              }
            ],
            "Free": 14,
            "Id": "DefaultRack",
            "Max": 14
          }
        ]
      }
    ],
    "Free": 24,
    "Max": 28,
    "layouts": [
      {
        "collection": "",
        "replication": "000",
        "writables": [
          1,
          2,
          3,
          4
        ]
      }
    ]
  },
  "Version": "0.47"
}
```



