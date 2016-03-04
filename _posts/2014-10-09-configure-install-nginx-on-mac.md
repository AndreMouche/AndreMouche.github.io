---
layout: post
title: "MAC上编译安装nginx"
keywords: ["nginx", "install"]
description: "configure and install nginx on mac"
category: "nginx"
tags: ["nginx","mac"]
comments: true
---

## 安装PCRE


可以在[官网](http://www.pcre.org/)下载pcre最新版。

```
tar zxvf pcre-8.36.tar.bz2 
cd pcre-8.36
sudo ./configure --prefix=/usr/local
sudo make 
sudo make install
```

## 安装Nginx

在[Nginx官网](http://nginx.org/)下载Nginx 最新版

```
tar zxvf nginx-1.6.2.tar.gz
cd nginx-1.6.2
./configure
```

### 编译概要

```
Configuration summary
  + using system PCRE library
  + OpenSSL library is not used
  + md5: using system crypto library
  + sha1: using system crypto library
  + using system zlib library

  nginx path prefix: "/usr/local/nginx"
  nginx binary file: "/usr/local/nginx/sbin/nginx"
  nginx configuration prefix: "/usr/local/nginx/conf"
  nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
  nginx pid file: "/usr/local/nginx/logs/nginx.pid"
  nginx error log file: "/usr/local/nginx/logs/error.log"
  nginx http access log file: "/usr/local/nginx/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"

```

由于直接使用默认参数make时出现MD5算法不能通过错误:

```
src/core/ngx_crypt.c:82:5: error: 'MD5_Init' is deprecated: first deprecated in OS X 10.7 [-Werror,-Wdeprecated-declarations]
    ngx_md5_init(&md5);
```

参考[此处](http://www.bingfengsa.com/info/31397.html)，即这里不能使用系统默认的crypto，使用ssl模块取代之，即在编译时加入以下参数

```
./configure --prefix=/usr/local --with-http_ssl_module --with-cc-opt="-Wno-deprecated-declarations"
sudo make
sudo make install
```

### 编译概要

```
Configuration summary
  + using system PCRE library
  + using system OpenSSL library
  + md5: using OpenSSL library
  + sha1: using OpenSSL library
  + using system zlib library

  nginx path prefix: "/usr/local"
  nginx binary file: "/usr/local/sbin/nginx"
  nginx configuration prefix: "/usr/local/conf"
  nginx configuration file: "/usr/local/conf/nginx.conf"
  nginx pid file: "/usr/local/logs/nginx.pid"
  nginx error log file: "/usr/local/logs/error.log"
  nginx http access log file: "/usr/local/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"

```

## 启动服务
```
[wuxuelian@fun:~/study/nginx/software/nginx-1.6.2]$ sudo /usr/local/sbin/nginx 
[wuxuelian@fun:~/study/nginx/software/nginx-1.6.2]$ ps aux | grep nginx
wuxuelian       36319   0.0  0.0  2432784    600 s001  S+    8:28PM   0:00.00 grep nginx
nobody          36316   0.0  0.0  2453500    908   ??  S     8:27PM   0:00.00 nginx: worker process
root            36315   0.0  0.0  2445088    340   ??  Ss    8:27PM   0:00.00 nginx: master process /usr/local/sbin/nginx
[wuxuelian@fun:~/study/nginx/software/nginx-1.6.2]$ sudo /usr/local/sbin/nginx -s stop
[wuxuelian@fun:~/study/nginx/software/nginx-1.6.2]$ ps aux | grep nginx
wuxuelian       36323   0.0  0.0  2442000    628 s001  S+    8:28PM   0:00.00 grep nginx
```

打开浏览器访问localhost:
![Alt welcome_to_nginx.png](/images/welcome_to_nginx.png)
## 停止服务

```
[wuxuelian@fun:~/study/nginx/software/nginx-1.6.2]$ ps aux | grep nginx
wuxuelian       36319   0.0  0.0  2432784    600 s001  S+    8:28PM   0:00.00 grep nginx
nobody          36316   0.0  0.0  2453500    908   ??  S     8:27PM   0:00.00 nginx: worker process
root            36315   0.0  0.0  2445088    340   ??  Ss    8:27PM   0:00.00 nginx: master process /usr/local/sbin/nginx
[wuxuelian@fun:~/study/nginx/software/nginx-1.6.2]$ sudo /usr/local/sbin/nginx -s stop
[wuxuelian@fun:~/study/nginx/software/nginx-1.6.2]$ ps aux | grep nginx
wuxuelian       36323   0.0  0.0  2442000    628 s001  S+    8:28PM   0:00.00 grep nginx
```

设置环境变量：

```
export PATH=/usr/local/bin/:/usr/local/sbin/:$PATH
```

启动Nginx

```
 sudo nginx 
```

需要停止Nginx的时候运行

```
 sudo nginx -s stop
``` 

参考：[在Mac OS X 10.9上编译安装nginx](http://www.kazaff.me/2013/07/18/%E5%9C%A8mac-os-x-10-9%E4%B8%8A%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85nginx/)
