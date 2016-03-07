---
layout: post
title: "Linux下C类程序性能测试必备技能"
keywords: ["c", "linux"]
description: ""
category: "linux"
tags: ["linux"]
comments: true
---

## 目录
 
 <div id="wmd-preview-section-24" class="wmd-preview-section preview-content">

</div><div id="wmd-preview-section-11400" class="wmd-preview-section preview-content">

<div><div class="toc"><div class="toc">
<ul>
<li><a href="#linux下c类程序性能测试必备技能">Linux下C类程序性能测试必备技能</a><ul>
<li><a href="#背景">背景</a></li>
<li><a href="#使用gdb调试程序">使用gdb调试程序</a></li>
<li><a href="#free查看当前内存使用情况">free查看当前内存使用情况</a></li>
<li><a href="#查看句柄数">查看句柄数</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</div>
</div></div>


# Linux下C类程序性能测试必备技能

## 背景

最近在玩c++的server端程序的开发，搞完后进行性能测试的时候，发现一晚上统计的内存一直程序上涨，用vargind工具扫了半天愣是没看到个definitely lost,想着项目月底要上线，瞬间慌乱无比，抓来性能测试小妹一顿敲诈勒索，小妹轻松搞定并传授相关技能如下。

## 使用gdb调试程序

* 正常启动 程序 exampleServer
*  ps获取进程号，若该程序涉及到多个子进程，则根据进程关系找到主进程。

```
// 获取tobie各进程间的关系，获取到当前exampleServer的主进程号PID
   ps -ef | grep exampleServer 
```

* 启动gdb调试当前项目 


```
//启动gbd调试PID
gdb attach $PID   
//进入gdb命令行后，c 运行当前程序
c       
//使用浏览器等客户端向exampleServer发送请求 －－其它案例中，只要想办法让exampleServer运行即可
// 在gbd命令行中
Ctrl＋C 停止服务
p 变量名  －－查看变量信息
```

## free查看当前内存使用情况
free 是个很好的命令

```
free -m
             total       used       free     shared    buffers   cached
Mem:        129179       2183     126996          0         15       1331
-/+ buffers/cache:        836     128342
Swap:         4094          0       4094
```

注意，used包含了cached部分
 我的问题就是出现在这里，由于程序运行过程中会产生大量的临时文件，虽然临时文件很快被删除，但还是会进入缓存，故运行过程中，缓存空间会持续上涨。在统计的时候，直接统计了used部分，故会持续上涨。这类情况对于本程序来说，属于正常现象，当缓存涨到一定程度，自然会持平，也就是说并不存在所谓的内存泄漏问题（测试小妹给力啊）。

## 查看句柄数

查看进程打开的fd个数

```
cd /proc/$PID
ls -lth
```

