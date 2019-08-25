---
layout: post
date: "2019-07-05 19:33:31"
title:  "Linux路由的相关配置和说明"
categories: Linux工具
tags: route
author: 肖邦
---

* content
{:toc}

> 平时工作中很多的网络问题都会与路由有些关系，搞清楚 Linux 系统中的路由概念和相关操作，对网络的理解以及对平时的开发工作、故障诊断都有很大的帮助。




## 一、Linux内核的路由表

路由表的作用就是指定下一级的网关，通过 `route` 可以查看 Linux 内核的路由表：

```bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.211.102.254  0.0.0.0         UG    0      0        0 eth1
10.211.102.0    0.0.0.0         255.255.255.0   U     0      0        0 eth1
169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eth0
192.168.11.0    192.168.11.253  255.255.255.0   UG    0      0        0 bond0.kni
```

`Gateway` 为 `0.0.0.0` 表示当前记录对应的 Destination 跟本机在同一网段，通信时不需要经过网关。下边总结下 Flags 的各种值的含义：

* `U` ：表示路由当前为启动状态（Up）
* `H` ：表示此网关为一主机（Host）
* `G` ：表示此网关为一路由（Gateway）
* `R` ：表示使用动态路由重新初始化的路由。
* `D` ：此路由是动态性地写入。
* `M` ：此路由是由守护程序或导向器动态修改。
* `!` ：表示此路由当前为关闭状态。

## 二、Linux内核的路由种类

* **主机路由**。路由表中指向单个 IP 地址或主机名的路由记录，其 Flags 字段为 H。下面示例中，对于 `10.0.0.10` 这个主机，通过网关 `10.139.128.1` 网关路由：

  ```bash
  $ route -n
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  10.0.0.10       10.139.128.1    255.255.255.255 UGH   0      0        0 eth0
  ...
  ```

* **网络路由**。主机可以到达的网络。下面示例中，对于 `10.0.0.0/24` 这个网络，通过网关 `10.139.128.1` 网关路由：

  ```bash
  $ route -n
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  10.0.0.0        10.139.128.1    255.255.255.0   UG    0      0        0 eth0
  ```

* **默认路由**。当目标主机的 IP 地址或网络不在路由表中时，数据包就被发送到默认路由（默认网关）上。默认路由的 `Destination`是 default 或 `0.0.0.0`。

  ```bash
  $ route
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  default         gateway         0.0.0.0         UG    0      0        0 eth0
  ```

## 三、路由的相关操作

* **添加主机路由**。

  ```bash
  $ route add -host 10.0.0.10 gw 10.139.128.1 dev eth0
  ```

* **添加网络路由**。

  ```bash
  $ route add -net 10.0.0.0 netmask 255.255.255.0 gw 10.139.128.1 dev eth0
  # or
  $ $ route add -net 10.0.0.0/24 gw 10.139.128.1 dev eth0
  ```

* **添加同一个局域网的主机**。不指定 gw 选项时，添加的路由记录不使用网关：

  ```bash
  $ route add -net 224.0.0.0 netmask 240.0.0.0 dev eth0
  ```

* **添加屏蔽路由**。

  ```bash
  $ route add -net 224.0.0.0 netmask 240.0.0.0 reject
  ```

* **删除可用路由**。

  ```bash
  $ route del -net 224.0.0.0 netmask 240.0.0.0
  ```

* **删除屏蔽路由**。

  ```bash
  $ route del -net 224.0.0.0 netmask 240.0.0.0 reject
  ```

* **添加和删除默认网关**。

  ```bash
  # 添加默认网关
  $ route add default gw 192.168.1.1
  # 删除默认网关
  $ route del default gw 192.168.1.1
  ```

* **静态路由永久生效**。平时我们可以将这些 route 命令放在 rc.local 文件中，其实可以有更专业的保存方式：

  ```bash
  # route add -net 11.1.1.0 netmask 255.255.255.0 gw 11.1.1.1
  # 保存在文件：/etc/sysconfig/static-routes 中，文件需要自己创建
  any net 11.1.1.0 netmask 255.255.255.0 gw 11.1.1.1
  ```

## 四、策略路由实现多默认路由

一般地说，在Linux系统路由表内只能有一条默认路由。当出站数据包根据目的IP地址选路失败后，会执行默认路由，交下一跳路由器（默认网关）转发数据包。现需要同时存在两条默认路由，数据包通过何种默认路由，由程序指定，数据包通过特定的路由规则转发到对应的路由器。

* 创建路由表

  ```bash
  $ echo "10 eth1table" >> /etc/iproute2/rt_tables
  $ echo "20 eth2table" >> /etc/iproute2/rt_tables
  ```

* 配置路由表，添加默认路由

  ```bash
  # 本机与默认网关的路由，否则会显示路由不可达
  $ ip route add 192.168.1.0/24 dev eth1 table eth1table
  $ ip route add 192.168.2.0/24 dev eth2 table eth2table
  # 默认网关
  $ ip route add default via 192.168.1.254 table eth1table
  $ ip route add default via 192.168.2.254 table eth2table
  ```

* 配置策略路由

  ```bash
  $ ip rule add from 192.168.1.1/32 table eth1table
  $ ip rule add from 192.168.2.1/32 table eth2table
  ```

