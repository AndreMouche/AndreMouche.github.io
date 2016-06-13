---
layout: post
title: "[译]weed-fs api(Volume Server)"
keywords: ["weed-fs", "storage"]
description: "api of weed-fs"
category: "weed-fs"
tags: ["weed-fs","storage"]
date: 2015-02-04
comments: true
---

## API OF WEED-FS(Volume Server)
原文链接[volume-server](http://weed-fs.readthedocs.org/en/latest/api.html#volume-server)

所有接口均可通过使用*&pretty=y*查看格式化的json串结果

###Upload File(上传文件)

```
curl -F file=@/home/chris/myphoto.jpg http://127.0.0.1:8080/3,01637037d6
{"size": 43234}
```
返回的size是上传到Seaweed-FS上的实际文件存储空间大小，有时候文件会根据mime type 被自动压缩。

### Upload File Directly(直接上传文件)

```
curl -F file=@/home/chris/myphoto.jpg http://localhost:9333/submit
{"fid":"3,01fbe0dc6f1f38","fileName":"myphoto.jpg","fileUrl":"localhost:8080/3,01fbe0dc6f1f38","size":68231}
```
该接口是为了方便用户上传。Master Server会得到一个file id,并将该文件存于正确的volume Server。
因该接口是为了简化上传，故当分配file id时并不支持各个参数（或者你可以加上并提交给我们代码）

### Delete File

```
curl -X DELETE http://127.0.0.1:8080/3,01637037d6
```

###Create а specific volume on a specific volume server(在制定volume Server上创建指定volume)

 创建3个volume在制定volume server上
 
```
curl "http://localhost:8080/admin/assign_volume?replication=000&volume=3"
```

若使用其它复制类型，如001，那么需要在其它volume server上执行相同创建请求，以创建镜像volume

###Check Volume Server Status

```
curl "http://localhost:8080/status?pretty=y"
{
  "Version": "0.34",
  "Volumes": [
    {
      "Id": 1,
      "Size": 1319688,
      "RepType": "000",
      "Version": 2,
      "FileCount": 276,
      "DeleteCount": 0,
      "DeletedByteCount": 0,
      "ReadOnly": false
    },
    {
      "Id": 2,
      "Size": 1040962,
      "RepType": "000",
      "Version": 2,
      "FileCount": 291,
      "DeleteCount": 0,
      "DeletedByteCount": 0,
      "ReadOnly": false
    },
    {
      "Id": 3,
      "Size": 1486334,
      "RepType": "000",
      "Version": 2,
      "FileCount": 301,
      "DeleteCount": 2,
      "DeletedByteCount": 0,
      "ReadOnly": false
    },
    {
      "Id": 4,
      "Size": 8953592,
      "RepType": "000",
      "Version": 2,
      "FileCount": 320,
      "DeleteCount": 2,
      "DeletedByteCount": 0,
      "ReadOnly": false
    },
    {
      "Id": 5,
      "Size": 70815851,
      "RepType": "000",
      "Version": 2,
      "FileCount": 309,
      "DeleteCount": 1,
      "DeletedByteCount": 0,
      "ReadOnly": false
    },
    {
      "Id": 6,
      "Size": 1483131,
      "RepType": "000",
      "Version": 2,
      "FileCount": 301,
      "DeleteCount": 1,
      "DeletedByteCount": 0,
      "ReadOnly": false
    },
    {
      "Id": 7,
      "Size": 46797832,
      "RepType": "000",
      "Version": 2,
      "FileCount": 292,
      "DeleteCount": 0,
      "DeletedByteCount": 0,
      "ReadOnly": false
    }
  ]
}
```




