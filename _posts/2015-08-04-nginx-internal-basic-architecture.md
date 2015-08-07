---
layout: post
title: "Nginx学习笔记(Nginx基础架构)"
keywords: ["nginx"]
description: "nginx basic architecture"
category: "nginx"
tags: ["nginx"]

---


##深入Nginx--Nginx基础架构


* 参考材料：《深入理解nginx模块开发与架构设计》 第三部分，深入Nginx-nginx基础架构

#目录
 <div id="wmd-preview-section-24" class="wmd-preview-section preview-content">

</div><div id="wmd-preview-section-11400" class="wmd-preview-section preview-content">

<div><div class="toc"><div class="toc">
<ul>
<li><a href="#源码目录结构">源码目录结构 </a></li>
<li><a href="#nginx的架构设计">Nginx的架构设计</a><ul>
<li><a href="#优秀的模块化设计">优秀的模块化设计</a></li>
<li><a href="#事件驱动架构">事件驱动架构</a></li>
<li><a href="#请求的多阶段异步处理">请求的多阶段异步处理</a></li>
<li><a href="#管理进程、多工作进程的设计">管理进程、多工作进程的设计</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</div>
</div></div>


## 源码目录结构 
```
.
├── auto            自动检测系统环境以及编译相关的脚本
│   ├── cc          关于编译器相关的编译选项的检测脚本
│   ├── lib         nginx编译所需要的一些库的检测脚本
│   ├── os          与平台相关的一些系统参数与系统调用相关的检测
│   └── types       与数据类型相关的一些辅助脚本
├── conf            存放默认配置文件，在make install后，会拷贝到安装目录中去
├── contrib         存放一些实用工具，如geo配置生成工具（geo2nginx.pl）
├── html            存放默认的网页文件，在make install后，会拷贝到安装目录中去
├── man             nginx的man手册
└── src             存放nginx的源代码
    ├── core        nginx的核心源代码，包括常用数据结构的定义，以及nginx初始化运行的核心代码如main函数
    ├── event       对系统事件处理机制的封装，以及定时器的实现相关代码
    │   └── modules 不同事件处理方式的模块化，如select、poll、epoll、kqueue等
    ├── http        nginx作为http服务器相关的代码
    │   └── modules 包含http的各种功能模块
    ├── mail        nginx作为邮件代理服务器相关的代码
    ├── misc        一些辅助代码，测试c++头的兼容性，以及对google_perftools的支持
    └── os          主要是对各种不同体系统结构所提供的系统函数的封装，对外提供统一的系统调用接口
```

## Nginx的架构设计

###优秀的模块化设计

高度的模块化设计是nginx的架构基础。在Nginx中，除了少量的核心代码，其它一切皆为模块。特点如下：

**高度抽象的模块接口**

所有的模块都遵循着同样的ngx_module_t接口设计规范，故具备以下特点：

* 良好的简单性
* 静态可扩展性
* 可重用性

**模块接口非常简单，具有很高的灵活性**

模块的基本接口nginx_module_t足够简单，只涉及到：

* 模块的初始化
* 模块的退出
* 对配置项的处理

这使得其具有足够的灵活性，使得nginx比较简单的实现了动态可修改性，即保持服务正常运行的情况下使系统功能发生改变。


```
struct ngx_module_s {
    //分类的模块的索引位置，
    //nginx 的模块可以分为四种:core,event,http和mail 
    //每个模块都会各自建立索引，ctx_index就是每个模在其所属类组的索引
    ngx_uint_t            ctx_index;
    //当前模块在 ngx_modules 里面的索引
    ngx_uint_t            index;

    //预留成员，目前尚未使用
    ngx_uint_t            spare0;
    ngx_uint_t            spare1;
    ngx_uint_t            spare2;
    ngx_uint_t            spare3;

    // Nginx模块版本
    ngx_uint_t            version;

    // 模块上下文，不同种类的模块有不同的上下文，因而实现了四种结构体
    void                 *ctx;
    //命令定义地址
    //模块的指令集，
    //每一个指令在源码中对应着一个ngx_command_t结构体变量
    ngx_command_t        *commands;
    // 模块的种类，用于区分core,event,http和mail 
    ngx_uint_t            type;

    // 初始化master时执行
    ngx_int_t           (*init_master)(ngx_log_t *log);

    // 初始化module时执行
    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    // 初始化process时执行
    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    // 初始化thread时执行
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    // 退出thread时执行
    void                (*exit_thread)(ngx_cycle_t *cycle);
    // 退出process时执行
    void                (*exit_process)(ngx_cycle_t *cycle);
        // 退出master时执行
    void                (*exit_master)(ngx_cycle_t *cycle);

    // 以下预留成员尚未使用
    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};
  
```


其中ngx_command_t结构如下：


```
struct ngx_command_s {
    // 指令名称的字符串，不可以包括空格
    ngx_str_t             name;
    // 用于设置指令在配置文件的哪一部分使用是合法的可选值
    ngx_uint_t            type;
    // 函数指针，这个函数主要是从配置文件中把该指令的参数(存放在 ngx_conf_t 中)
    // 转换为合适的数据类型，并将转换后的值保存到模块的配置结构体中
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    // 用于指示配置项所处内存的相对偏移位置。对于http模块，conf的设置是必要的。
    // http在调用指令解析函数时，自动将解析出的配置项写入到http模块代码定义的
    // 结构体中，在配置项简介中了解到，http模块可能定义3个结构体，分别存储于
    // main,srv,loc级别的配置项中。（对应于create_main_conf,create_srv_conf,
    // create_loc_conf方法创建的结构体），而http框架在自动解析时需要知道应把
    // 解析出的配置项值写入到哪个结构体中。这点需要依赖conf配置的值
    ngx_uint_t            conf;
    //表示当前配置项在整个存储配置项的结构体中的偏移位置，以字节为单位。举个例子：
    //我们通过指令函数回调时，需要通过conf成员找到应该用哪个结构体存放，然后
    //通过offset成员找到这个结构体中的相应成员，以便存放该配置。
    ngx_uint_t            offset;
    // 用来辅助指令回调函数，可以使其更加灵活
    void                 *post;
};

```

**配置模块的设计**

Nginx的配置模块的类型（ngx_module_s中的type）叫做NGX_CONF_MODULE，它仅有的模块叫做ngx_conf_module,是nginx的最底层模块，它指导着所有模块以配置项为核心来提供功能。因此它是所有模块的基础。配置模块为nginx提供了以下特性：

* 高可配置性
* 高可扩展性
* 高可定制性
* 高可伸缩性


**核心模块接口的简单化**

核心模块的类型叫做NGX_CORE_MODULE。目前官方的核心类型模块中共有6个具体模块：

* ngx_core_module
* ngx_errlog_module
* ngx_events_module
* ngx_openssl_module
* ngx_http_module
* ngx_mail_module

这些核心模块简化了nginx的设计，使得非模块化的框架代码只关注于如何调用6个核心模块

核心模块的接口定义如下：

```
 typedef struct {
     //核心模块名称
     ngx_str_t             name;
     //解析配置项前，Nginx框架会调用create_conf方法
     void               *(*create_conf)(ngx_cycle_t *cycle);
     //解析配置项后，Nginx框架会调用init_conf方法
     char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
 } ngx_core_module_t;
```

ngx_module_t接口及其对核心、事件、HTTP、mail等四类模块ctx上下文成员的具体化结构如下：
<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/nginx/ngx_module_t.jpg?raw=true" alt="ngx_module_t.jpg" title="ngx_module_t.jpg" width="600" />

**多层次、多类别的模块设计**

所有的模块间时分层次、分类别的，官方Nginx共有五大类型的模块：

* 核心模块
* 配置模块
* 事件模块
* HTTP模块
* mail模块

详细层次结构如下图

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/nginx/nginx_core_module.jpg?raw=true" alt="ngx_module_t.jpg" title="ngx_module_t.jpg" width="600" />

其中

* 配置和核心模块是由Nginx的框架代码定义的，是其它所有模块的基础。
* 配置模块是所有模块的基础，它实现了最基本的配置项解析功能。
* Nginx框架会调用核心模块，但是其它三种模块都不会与框架产生关系
* 事件、HTTP、mail这三种模块的共性是：他们在核心模块中拥有自己的代言人，并在同类模块中有一个作为核心业务与管理功能的模块
* 事件模块由ngx_event_module定义，但所有事件模块的加载则由ngx_event_core_module负责
* http模块由ngx_http_module定义，并由它负责加载所有的http模块，而ngx_http_core_module则负责业务的核心逻辑、决定处理具体的请求的具体http模块。mail模块与http类似
* 事件模块是http模块和mail模块的基础


###事件驱动架构
事件驱动架构，即由一些事件发生源来产生事件，由一个或多个事件收集器来收集、分发事件，然后许多事件处理器会注册自己感兴趣的事件，同时会消费这个事件。

**传统Web服务器处理事件的简单模型**

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/nginx/traditional_web_event_ar.jpg?raw=true" alt="ngx_module_t.jpg" title="traditional_web_event_ar.jpg" width="600" />

**Nginx处理事件的简单模型**

<img src="https://github.com/AndreMouche/AndreMouche.github.io/blob/master/images/nginx/nginx_process_events_ar.jpg?raw=true" alt="nginx_process_events_ar.jpg" title="nginx_process_events_ar.jpg" width="600" />

传统的Web服务器是每个事件消费独占一个进程资源，Nginx的事件消费者只是被事件分发者进程短期调用而已。
其中nginx的这种设计优劣如下：

**优点**

* 使得网络的性能、用户感知的请求时延（延时性）都得到了提升
* 每个用户的请求所产生的事件会及时响应
* 整个服务器的网络吞吐量会由于事件的及时响应而增大

**缺点**

* 每个事件消费者都不能有阻塞行为，否则将会由于长时间占用事件分发者进程而导致其它事件得不到及时响应
* 每个消费者不能让进程变为休眠或等待状态，如在等待一个信号量条件的满足时会使进程进入休眠状态
* 加大了消费事件程序的开发难度

###请求的多阶段异步处理

请求的多阶段异步处理，即把一个请求的处理过程按照事件的触发方式划分为多个阶段，每个阶段都可以由事件收集、分发器来触发。

另：请求的多阶段异步处理职能基于事件驱动架构实现。

示例：

**处理获取静态文件的HTTP请求时切分的阶段及各阶段的触发事件**

|阶段意义|触发事件|
|:--|:--|
|建立TCP连接|接收到TCP中的SYNC包|
|开始接收用户请求|接收到TCP中的ACK包表示建立连接成功|
|接收到用户请求并分析已接收的请求是否完整|接收到用户的数据包|
|接收到完整的用户请求后开始处理用户请求|接收到用户的数据包|
|由目标静态文件中读取部分内容（避免长期阻塞事件分发者进程）并直接发送给用户|接收到用户的数据包；或者接收到TCP的ACK包表示用户已接收到上次发送的数据包，TCP滑动窗口向前滑动|
|对于非Keep-alive请求，发送完静态文件后主动关闭连接|接收到TCP中的ACK包表示拥护已接收到之前发送的所有数据包|
|由于用户关闭连接而结束请求|接收到TCP中的FIN包|

这七个阶段是可以重复发生的，即当一个下载静态资源请求可能会由于请求数据过大、网速不稳定等因素而被划分为成百上千个上述阶段。

**多阶段异步处理的优势如下**

* 配合事件驱动的架构，将会极大地提高网络性能
* 使得每个进程都能全力运转，不会或者尽量少地出现进程休眠状况。
* ...


**划分请求阶段的原则**

一般是找到请求处理流程中的阻塞方法（或者造成阻塞的代码段），在阻塞代码段上按照下面4种方式来划分阶段

1. 将阻塞进程的方法按照相关的触发事件分为两个阶段：如send调用发送数据给用户时，分为两个阶段：发送且不等待结果阶段、send结果返回阶段
2. 将阻塞方法调用按照时间分解为多个阶段的方法调用：如读取10MB的文件，分为1000次，每次读取10KB
3. 在"无所事事" 且必须等待的系统的响应，从而导致系统空转时，使用定时器划分阶段。如那些循环检查标志位。。
4. 如果某个阻塞方法完全无法划分，则必须使用独立的进程执行这个阻塞方法


###管理进程、多工作进程的设计

Nginx采用一个master,多个worker工作进程的设计方式，优点如下：

* 利用多核系统的并发处理能力
* 负载均衡
* 管理进程会负责监控工作进程的状态，并负责管理其行为


