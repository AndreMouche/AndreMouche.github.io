---
layout: post
title: "BPG调研"
keywords: ["BPG", "Graphics"]
description: "information of bpg"
category: "Graphics"
tags: ["BPG"]
comments: true
---

# BPG调研

## 目录 


<ul>
<li>简介</li>
<li>工具 
<ul>
<li>兼容性</li>
<li>安装</li>
<li>效果示例</li>
</ul>
</li>

<li>性能
<ul>
<li>BPG编解码性能</li>
<li>与WEBP对比</li>
</ul>
</li>
</ul>



本文基于libbpg-0.9.5展开调研

## 简介 

BPG是一种新型的图片格式。其设计初衷在于当图片质量或文件size成为瓶颈时，取代JPEG。其主要特点如下：

1. 高压缩比。BPG在quality类似的情形下，比JPEG要小得多。相同大小的图片，使用BMP存储质量远高于JPEG
2. 浏览器支持：使用一个很小js解码库（54KB）后，便可被大部分的浏览器所支持。
3. 算法：基于HEVC开源的标准视频压缩算法的一个子集实现
4. 支持与jpeg相同的chroma格式用于减少压缩过程中的数据丢失。支持alpha通道。支持 RGB, YCgCo和CMYK色彩空间。
5. 更大的通道范围支持：原生支持8位和16位通道。
6. 支持无损压缩
7. 支持多种元数据（如EXIF，ICC profile,XMP）
8. 支持动画


## 工具

### 兼容性

bpg作为一种新型图片格式，对其支持的工具尚不健全。除了[官方](http://bellard.org/bpg/)[推荐工具](http://bellard.org/bpg/libbpg-0.9.5.tar.gz)可用外，像GraphicsMagick、ffmpeg,exiftool等主流多媒体处理软件尚未对其支持。

**系统支持**

bpg官方发布的工具支持三种系统，具体情况如下：

| 系统 | 编码 |解码 |查看|备注|
|:----:|-----:|:---:|:-------:|:---:|
|Winows  |bpgenc支持jpg/png=>bpg|bpgdec支持bpg=>png,<br/>但不支持自定义输出文件|bpgview支持查看|~|
|Mac OS |bpgenc支持png=>bpg|bpgdec支持bpg=>png,<br/>但不支持自定义输出文件|没有查看工具|~|
|Linux  |bpgenc支持png/jpg=>bpg|bpgdec支持bpg=>png|没有查看工具|~|

**浏览器支持**

WEBP浏览器：需带上js解码库。

### 安装

**windows**

直接下载[bpg-0.9.5-win32.zip](http://bellard.org/bpg/bpg-0.9.5-win32.zip)即可使用

**MacOS**

[brew install](http://brew.sh/)

**Ubuntu**

1. 安装libpng16以上版本(编译安装)
2. 安装libjpeg62*(apt-get install) 
3. 安装libsdl-image*(apt-get install)
4. 编译安装[libbpg-0.9.5.tar.gz](http://bellard.org/bpg/libbpg-0.9.5.tar.gz)


### 效果示例 

左边是BPG 效果而右边是JPG
![](http://cdn.unwire.hk/wp-content/uploads/2014/12/comparison.jpg)

示例地址:[http://xooyoozoo.github.io/yolo-octo-bugfixes/#zoo-bird-head&jpg=s&bpg=s](http://xooyoozoo.github.io/yolo-octo-bugfixes/#zoo-bird-head&jpg=s&bpg=s)

**web端显示示例**

需要引用对应javascript库：
[http://bellard.org/bpg/bpgdec8a.js](http://bellard.org/bpg/bpgdec8a.js)

完整的html示例


## 性能

### BPG编解码性能

测试场景：

1.  数据：线上1000张图片转换而得的各类格式
2.  环境：Ubuntu 14.04.1 LTS (GNU/Linux 3.13.0-32-generic x86_64)
3.  工具：[libbpg-0.9.5.tar.gz](http://bellard.org/bpg/libbpg-0.9.5.tar.gz)

| 源图 | 目标图 |数量 |失败次数|源大小|结果大小|空间变化(dest-src)/src| 耗时(s)|
|----:|-----:|---:|-------:|------:|-------:|-------------:|-------:|
|JPG  |BPG   |1000|0	   |15M	   |8.1M    |-46%		   |1762    |
|PNG  |BPG   |1000|0       |103M   |8.1M	|-92.1%	       |1765    |
|BPG  |PNG   |1000|0       |8.1M   |96M		|+1085%		   |92      |

结论：
1. 同一张图片内容，JPG,PNG转为BPG后空间显著变小。
2.  PNG可与BPG自由转换，任意JPG可成功转为BPG。
3.  PNG/JPG转换为BPG耗时较长，BPG转回PNG相对较快。

### 与WEBP对比

webp测试场景
1.  数据：线上1000张图片转换而得的各类格式
2.  环境：Ubuntu 14.04.1 LTS (GNU/Linux 3.13.0-32-generic x86_64)
3.  工具：[libwebp 0.4.1 ](http://downloads.webmproject.org/releases/webp/libwebp-0.4.1-linux-x86-32.tar.gz)

| 源图 |目标图 |数量 |失败次数|源大小|结果大小|空间变化(dest-src)/src| 耗时(s)|
|----:|-----:|---:|-------:|------:|-------:|-------------:|-------:|
|JPG  |WEBP  |1000|0	   |15M    |9.6M	|-36%		   |58		|
|JPG  |BPG   |1000|0	   |15M	   |8.1M    |-46%		   |1762    |
|PNG  |WEBP  |1000|0       |103M   |9.8M    |-91%          |66		|
|PNG  |BPG   |1000|0       |103M   |8.1M	|-92.1%	       |1765    |
|WEBP |PNG   |1000|0	   |9.7M    |103M	|+962%		   |80		|
|BPG  |PNG   |1000|0       |8.1M   |96M		|+1085%		   |92      |

结论：
1.相比BPG，WEBP的压缩效果相对差一些，但转换速度快得多。











