---
layout: post
title: "配置VirtualBox上UbuntuServer的网络"
keywords: ["Mac", "network "]
description: "network config on virtual server"
category: "NetWork"
tags: ["network","mac"]

---


在Mac OS 的VirtualBox上安装Ubuntu Server，使得其网络满足以下需求：

1. 能连接到外网
2. mac 终端能ssh 到虚拟机

##安装Ubuntu Server

    ubuntu-14.04.1-server-amd64.iso 

##设置虚拟机配置
server=>Config打开虚拟机配置项

NAT:

<img src="/images/adapter1.png" alt="NAT" title="nointerlace1.PNG" width="600" />


Host-only adapter

<img src="/images/adapter2.png" alt="NAT" title="Host-only adapter" width="600" />


Bridged Adapter

<img src="/images/adapter3.png" alt="NAT" title="Bridged Adapter" width="600" />


 
## 启动虚拟机，查看MAC 当前网络

看到如下行(是我之前配置的？不记得啦)：

```
vboxnet0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	ether 0a:00:27:00:00:00 
	inet 192.168.56.1 netmask 0xffffff00 broadcast 192.168.56.255
```

##虚拟机网络配置
登陆虚拟机，修改
/etc/network/interfaces

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp

# Virtualbox Host-only mode
auto eth1
iface eth1 inet static
address 192.168.56.190
netmask 255.255.255.0
network 192.168.56.0
broadcast 192.168.56.255
```

重启虚拟机，再次ifconfig：

```
fun@fun:~$ ifconfig
eth0      Link encap:Ethernet  HWaddr 08:00:27:3f:dc:1a  
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe3f:dc1a/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:30442 errors:0 dropped:0 overruns:0 frame:0
          TX packets:12600 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:27478206 (27.4 MB)  TX bytes:798887 (798.8 KB)

eth1      Link encap:Ethernet  HWaddr 08:00:27:90:d2:df  
          inet addr:192.168.56.190  Bcast:192.168.56.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe90:d2df/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:8343 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5120 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:3550804 (3.5 MB)  TX bytes:777058 (777.0 KB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)


```

Done

##说明：
学习Nginx时想尽量可能的接近线上环境去玩，所以才在虚拟机上装了Ubuntu-Server，但网络问题一直搞不定。
网络基础是在差得一塌糊涂，网上查了三种连接方式，说得最多的是桥接，但鉴于天资实在有限，一直搞不定。
今晚一怒之下，把三个adapter全配上了，莫名奇妙地可以了，这边只是简单做个笔记，mark一下。

哎，等哪天有空了一定得好好研究一下，补补网络知识，一定得知其所以然才行。