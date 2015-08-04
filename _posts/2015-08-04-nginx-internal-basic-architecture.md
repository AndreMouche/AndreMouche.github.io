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

