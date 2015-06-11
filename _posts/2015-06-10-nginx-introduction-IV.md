---
layout: post
title: "Nginx介绍(译文IV--深入nginx内核)"
keywords: ["nginx"]
description: "nginx"
category: "nginx"
tags: ["nginx"]

---


##Nginx 介绍（译文IV--深入nginx内核）

* 原文 [nginx](http://www.aosabook.org/en/nginx.html)
* 作者 [Andrew Alexeev](http://www.aosabook.org/en/intro2.html#alexeev-andrew)

##深入nginx内核
如前文所提及，nginx的源码主要包括一个核心（core）和多个模块(modules)。其中核心(core)部分主要用于提供以下功能：

* 作为web服务器的基础
* 提供web及邮箱的反向代理（mail reverse proxy）的功能
* 允许使用底层网络协议
* 创建必要的运行时环境
* 保证各个模块间的无缝交互

需要注意的是：大部分的协议和为应用程序定制的特性，由nginx的模块（modules）而非核心（core）完成。

在nginx的内部，通过各个模块间的管道(pipeline)或模块链(chain)来处理所有连接。或者说，对于每个操作，都有相应的模块在处理该工作，例如

* 压缩
* 修改内容
* 执行SSI（server-side includes）
* 通过FastCGI、uwsgi协议与后端应用服务器交互
* 与memcache交互。

在核心(core)与实际功能模块（real "functional" modules）之间，还有一对模块，即http和mail。这两个模块在核心(core)和更底层的组建中间提供了一个附加的抽象层。在这些模块中，处理与各自应用层协议相关的事件序列，如已实现的HTTP，SMTP，IMAP。


结合Nginx核心(core)，这些上层的模块负责维护调用不同功能模块的正确顺序。虽然目前HTTP协议是作为http模块的一部分实现的，但为支持其他协议如SPDY（参考“[SPDY: An experimental protocol for a faster web](http://www.chromium.org/spdy/spdy-whitepaper)），将http独立为一个功能模块已列入计划中。


功能模块可以分为以下几类：

* 事件模块
* 阶段处理器
* 输出过滤器
* 变量处理器
* 协议模块
* 上游及负载均衡器

虽然mail模块中用到了事件模块和协议，但以上大部分模块用于补充nginx的HTTP功能。

* 事件模块提供了类似于kqueue和epool的基于操作系统的事件通知机制，主要取决于操作系统的能力与编译配置。
* 协议模块允许nginx通过HTTPS, TLS/SSL, SMTP, POP3 和 IMAP等协议通信。

典型的HTTP请求处理周期如下：

1. 客户端发送HTTP请求。
2. nginx核心根据配置匹配该请求的location，选择对应的阶段处理器。
3. 根据配置需要，负载均衡器挑选一个上游服务器用于转发请求。
4. 阶段处理器完成工作，并将每个输出缓冲区传递给第一个过滤器。
5. 第一个过滤器将输出传给第二个过滤器。
6. 第二个过滤器传递输出给第三个（。。。）。
7. 最终将响应发送给客户端。

Nginx模块调用是高度可定制的。它主要通过一系列的回调展开工作，而这些回调则通过使用指向可执行函数的指针来实现。因而，对于那些想要自己编写模块的开发者，就必须准确地定义这些自定义模块如何运行、何时运行，从而大大加重了负担。为了缓解该负担，使之能够更好地执行，Nginx的API和开发者文档都在不断地优化中。

一些在nginx中插入模块的案例：

* 读取和处理配置文件之前
* Location和server的每个配置指令生效时
* Main配置被初始化后
* Server配置（host/port）初始化后
* Server配置合并到main配置后
* Location配置初始化或者被合并到父server配置时
* Master进程启动或退出时
* 新的worker进程启动或退出时
* 处理请求时
* 过滤响应头和响应体时
* 为request挑选，初始化和重新初始化上游服务器时
* 处理上游服务器响应时
* 完成与上游服务器的交互时

