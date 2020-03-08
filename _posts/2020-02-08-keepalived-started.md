---
layout: post
date: "2020-02-08 20:13:13"
title:  "keepalived 入门介绍"
categories: 负载均衡
tags: lvs
author: 肖邦
---

* content
{:toc}

> keepalived 是一个与 LVS 息息相关的软件，最初就是为 LVS 而生的。它提供了负载均衡和高可用的框架，负载均衡功能是依赖于 IPVS 内核模块，高可用功能是通过 VRRP 协议实现的，两个功能可以独立使用。




## LVS 基本操作方式

负载均衡 LVS 是章文嵩博士在 1998 年开源的项目，算是国内最早的开源项目。LVS 是一个高性能的负载均衡器，能够很容易的实现分布式集群。LVS 在内核是以模块 ko 的形式存在，运行基于 netfilter 框架之上。所以集群服务规则的配置与操作是通过 LVS 提供用户工具 ipvsadm 进行的，比如我们可以执行如下命令：

```bash
$ /sbin/modprobe ip_vs  # 加载 ipvs 模块

# 创建 TCP 80 的服务
$ ipvsadm -A -t 182.18.100.16:80 -s rr

# 为 TCP 80 添加 RS1 192.168.1.2
$ ipvsadm -a -t 182.18.100.16:80 -r 192.168.1.2:80 -g -w 1
# 为 TCP 80 添加 RS2 192.168.1.3
$ ipvsadm -a -t 182.18.100.16:80 -r 192.168.1.3:80 -g -w 1
# 这样就创建了一个 182.18.100.16:80 的虚拟服务器集群（后端可以是多个 RS）

# 如果 RS1 宕机，我们需要将 RS1 从几种中摘掉
ipvsadm -d -t 182.18.100.16:80 -r 192.168.1.2:80
```

我们可以通过 ipvsadm 工具来对负载均衡 LVS 进行各种服务变更的操作，包括 service 和 rs 的增删改查。ipvsadm 是 LVS 官方配套的工具，它是个用户态的应用程序，运行在用户空间，它是通过 `raw socket` 或 `netlink` 方式与内核 LVS 进行通信的，它与 LVS 之间的关系就类似 iptables 与 netfilter 之间的关系和位置。

```bash
用户：  ipvsadm
          |
          | (libipvs)
----------+----------------------------
          | (raw socket | netlink)
          ↓
内核：    LVS

# 注：ipvsadm 是 LVS 服务规则配置的工具，iptables 也是防火墙规则配置的用户工具。
```

## keepalived 解决的问题

从配置操作上来看，ipvsadm 工具能够实现 service 和 rs 增删改查，看起来完全能够胜任，在简单的业务场景下，使用上也特别大的问题。但是我们实际生产环境中，不会就如此简单。服务不会只有这么少，rs 数量也会非常多，变更、升级、维护等操作也会很频繁，如果只通过 ipvsadm 方式操作，会有很多缺点

* 配置不易管理
* 操作效率低
* 容易出错。
* **另外一个问题**：LVS 服务后端有多个 RealServer 服务器，若某个 RS 异常宕机，LVS 仍然会将流量转发给异常 RS，这样就会导致服务的部分流量异常，这是一个不能够接受的问题。

大概在 2001 年的时候，名叫 "Alexandre Cassen" 工程师开发了 keepalived 软件，大概专门为解决 LVS 配置管理问题。早期发布的 keepalived-0.2.x 版本支持配置文件以 "virtual_server" 的方式来配置服务规则。

keepalived 除了解决配置管理维护问题，还提供了健康检查的功能：
* 对每台 RS 服务器进行周期性的健康检查。
* 如果某个 RS 异常宕机，自动会把异常服务器从服务列表中摘除。
* 如果某个宕机的 RS 恢复正常后，也会自动把 RS 加入到服务列表中提供服务。

> 到目前为止，keepalvied 解决了配置管理和健康检查的问题。在使用上，我们只需要在 Linux 服务器上安装 keepalvied ，通过配置 keepalived.conf 配置文件，启动 keepalived 程序，就可以正常使用 LVS 的负载均衡功能了，看起来好像是 keepalived 实现了负载均衡功能。所以导致很多新手搞不清楚 keepalived 与 LVS 之间的关系，忽略 IPVS 的存在。


还存在另外一个问题，LVS 服务器存在单点故障的风险，也就是说 LVS 随时都有可能发生宕机故障，可恢复的时间存在不确定性，严重影响服务体验和可用性。keepalvied 在 0.4.x 版本中增加了高可用特性的支持，是通过 VRRP 协议实现的。我们使用两台 Linux 服务器分别 LVS + keepalived 环境，只有其中 1 台对外提供服务（配置 VIP），另外一台作为热备份，只有在 server1 服务器宕机或不可提供服务时，keepalived 会自动将 VIP 迁移到 server2 上，实现服务器的高可用。VIP 切换过程中是不需要人为干预的。


## keepalived 简单介绍

负载均衡是一种基于 IP 的流量转发，它可以提供多个虚拟的服务。在设计负载均衡拓扑时，负载均衡的可用性至关重要。

keepalived 本质上是提供了负载均衡和高可用的两个基础框架。负载均衡框架依赖于 IPVS 内核模块（提供 4 层负载均衡），并对此实现了多种方式的健康检查，能够动态的维护和管理后端的服务器的健康状况。高可用是通过 VRRP 协议实现的，keepalived 的两种框架也可以独立使用，互不依赖。它提供的高可用功能可以为其它应用程序，如 nginx、haproxy、MySQL 等提供噶可用的解决方案。


简单总结，keepalived 主要有如下 3 方面的功能：

* LVS 配置管理与操作
* LVS 后端服务器的健康检查
* 通过 vrrp 实现高可用功能


## keepalived 软件设计

keepalived 软件采用纯 C 进行编写的，该软件通过 IO 多路复用器提供了即时的网络设计。主要设计重点是所有组件之间的模块化，创建 core 库避免代码重复，实现安全可靠的代码，以确保生产环境的健壮性和稳定性。

**（一）进程模型**

为了确保健壮性和稳定性，keepalived 分为了 3 个进程：

* 主进程：fork 两个子进程后，主要负责监控 2 个子进程，起到 WatchDog 作用。
* vrrp 进程
* checker 进程


**（二）内核组件**

keepalived 内核涉及到了如下 4 个组件：

* `LVS Framework`：使用 getsockopt 和 setsockopt 方式与内核进行基础通信。
* `Netfilter Framework`：IPVS 负载均衡代码框架。
* `Netlink Interface`：在服务器网卡上设置 | 删除 VIP。
* `Multicast`：VRRP 宣告消息是通过保留的组播地址（224.0.0.18）进行的。

**（三）系统组件**

![keepalived](https://raw.githubusercontent.com/liwei0526vip/blogimg/master/lb008keepalived_software_design.png)

* **控制平面（Control Plane）**。keepalived 主配置文件是 keepalived.conf，配置解析器与配置项以树层次结构组成，每个配置项都有相应的 handler ，支持 include 方式，在解析期间，配置信息会被转换为内部数据结构存储。
* **调度器（I/O Multiplexer）**。所有的事件都是在进程内部执行的调度。它的 IO 模型使用的是 select，负责内部所有的任务。它并没有使用 Posix 线程，而是针对网络目的而优化的虚拟线程。
* **内存管理**。该组件提供了通用的内存管理功能，可以在两种模式下使用：正常模式和调试模式。 使用debug_mode时，它提供了一种强大的方法来消除和跟踪内存的泄漏。
* **核心组件库**。定义并提供了一些通用的、全局的库供系统程序使用。这些库包含：html 解析、like-list、timer、vector、字符串格式化、buffer dump、网络组件、daemon management、pid handing、TCP Layer4。
* **看门狗（WatchDog）**。主进程负责监控 vrrp 和 checker 进程的健康状况。
* **Checker**。是 keepalived 最重要的功能之一，检查 RS 健康状态，能够自动摘除和恢复后端服务器。
* **VRRP Stack**。是 keepalived 重要的功能，通过 VRRP 协议实现的高可用功能。
* **SystemCall**。提供了调用外部脚本的能力。
* **Netlink Reflector**。keepalived 具有自己的网络接口表示形式，IP 地址和接口标志是通过内核 netlink 设置和监视的，用于设置 VIP。另一方面，能够将任何与接口相关的事件同步到 keepalived 相关数据结构中，因此任何其他应用程序对接口的操作在 keepalived 内部都能感知得到。
* **SMTP**。主要是用于实现邮件通知机制。
* **IPVS Wrapper**。主要用于 keepalived 向 IPVS 代码下发拓扑规则，它提供了 keepalived 数据与 IPVS 数据的转换，它使用 libipvs 保持与 IPVS 的通用集成。
* **IPVS**。LVS 负载均衡的核心代码。
* **NETLINK**。Netlink 用于在内核和用户空间进程之间传输信息。 它由一个用于用户空间进程的基于套接字的标准接口和一个用于内核模块的内部内核  API 组成。
* **Syslog**。日志系统。

**（四）健康检查框架**

目前支持如下几种健康检查：

* TCP_CHECK
* HTTP_GET
* SSL_GET
* MISC_CHECK

**（五）Failover（vrrp）框架**

实现 FailOver 功能。


## keepalived 安装

**（一）、二进制安装方式**

```bash
$ yum install keepalived       # CentOS
$ apt-get install keepalived   # Ubuntu
```

**（二）、源码编译安装**

* 安装依赖
  ```bash
  $ yum install curl gcc openssl-devel libnl3-devel net-snmp-devel
  # or
  $ apt-get install curl gcc libssl-dev libnl-3-dev libnl-genl-3-dev libsnmp-dev
  ```
* 编译安装
  ```bash
  $ curl -O http://keepalived.org/software/keepalived-1.x.x.tar.gz
  $ tar zxf keepalived-1.x.x.tar.gz
  $ cd keepalived-1.x.x
  $ ./configure --prefix=/usr/local/keepalived-1.x.x
  $ make && make install
  ```
* 设置初始化脚本
  ```bash
  $ ln -s /etc/rc.d/init.d/keepalived.init /etc/rc.d/rc3.d/S99keepalived
  # or
  $ ln -s /etc/init.d/keepalived.init /etc/rc2.d/S99keepalived
  ```

## keepalived 配置模板

**（一）全局配置**

```
global_defs {
    notification_email {
        email
        email
    }
    notification_email_from email
    smtp_server host
    smtp_connect_timeout num
    lvs_id string
}
```

**（二）virtual server 配置**

```
virtual_server (@IP PORT)|(fwmark num) {
    delay_loop num
    lb_algo rr|wrr|lc|wlc|sh|dh|lblc
    lb_kind NAT|DR|TUN
    (nat_mask @IP)
    persistence_timeout num
    persistence_granularity @IP
    virtualhost string
    protocol TCP|UDP

    sorry_server @IP PORT
    real_server @IP PORT {
        weight num
        TCP_CHECK {
            connect_port num
            connect_timeout num
        }
    }
    real_server @IP PORT {
        weight num
        MISC_CHECK {
            misc_path /path_to_script/script.sh
            (or misc_path “ /path_to_script/script.sh <arg_list>”)
        }
    }
}
```

**（三）VRRP 配置**

```
vrrp_sync_group string {
    group {
        string
        string
    }
    notify_master /path_to_script/script_master.sh
        (or notify_master “ /path_to_script/script_master.sh <arg_list>”)
    notify_backup /path_to_script/script_backup.sh
        (or notify_backup “/path_to_script/script_backup.sh <arg_list>”)
    notify_fault /path_to_script/script_fault.sh
        (or notify_fault “ /path_to_script/script_fault.sh <arg_list>”)
}
vrrp_instance string {
    state MASTER|BACKUP
    interface string
    mcast_src_ip @IP
    lvs_sync_daemon_interface string
    virtual_router_id num
    priority num
    advert_int num
    smtp_alert
    authentication {
        auth_type PASS|AH
        auth_pass string
    }
    virtual_ipaddress { # Block limited to 20 IP addresses
        @IP
        @IP
        @IP
    }
    virtual_ipaddress_excluded { # Unlimited IP addresses
        @IP
        @IP
        @IP
    }
    notify_master /path_to_script/script_master.sh
        (or notify_master “ /path_to_script/script_master.sh <arg_list>”)
    notify_backup /path_to_script/script_backup.sh
        (or notify_backup “ /path_to_script/script_backup.sh <arg_list>”)
    notify_fault /path_to_script/script_fault.sh
        (or notify_fault “ /path_to_script/script_fault.sh <arg_list>”)
}
```

## keepalived 主程序相关参数

* -f：指定配置文件
* -P：只运行 VRRP 子系统，不运行健康检查子系统。
* -C：只运行 Checker 子系统，不运行 VRRP 子系统。
* -l：日志消息打印到本地控制台。
* -D：详细的日志消息。
* -S：设置 syslog 设备，LOG_LOCAL[0-7]，默认是 LOG_DAEMON
* -V：keepalived 进程退出时，不删除 VIP
* -I：keepalived 进程退出时，不删除 IPVS 的拓扑规则
* -R：不要重新生成子进程。
* -n：不要设置 daemon 守护进程，在前台运行。
* -d：dump 配置文件数据。
* -p：指定 keepalived 进程 ID 文件
* -r：指定 vrrp 进程的 pidfile
* -c：指定 checker 进程的 pidfile

启动守护进程：

```bash
$ /etc/rc.d/init.d/keepalived.init start
```

