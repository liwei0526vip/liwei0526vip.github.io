---
layout: post
date: "2020-02-10 20:26:11"
title:  "高性能负载均衡 DPVS 路由子系统的实现"
categories: 负载均衡
tags: lvs dpvs
author: 肖邦
---

* content
{:toc}
> Linux 网络栈最重要的目标之一就是转发流量，尤其对核心路由器来说更是如此。Linux 路由子系统负责转发数据包和维护转发数据库。本篇文章先简单介绍 Linux 系统下路由系统，然后总结 DPVS 中路由模块的实现。




## 一、路由的作用

路由系统位于 IP 层，主要是实现 IP 数据包的路由转发作用。
* 路由器。路由器可以隔离二层广播域，能够实现多个链路之间三层的互通，就是根据路由表项，将数据包转发到正确的链路。
* 服务器。Linux 服务器在发送数据包时，需要查找路由表项，确定从哪个网卡出去、确定 "下一跳" 等信息，才能够将数据包发送出去。

由上可知路由在 Linux 和路由器中的重要作用，而 DPVS 路由的查找和维护都是自己实现的，不过相比 Linux 而言较为简单了些。

## 二、Linux 路由系统


**（一）路由表**

Linux 系统中可以定义 1-252 个路由表，由系统本身维护了 4 个路由表：
* `0` - 系统保留
* `253` - default 表，没有指定的默认路由都放在这个表里
* `254` - main 表，没有指明路由表的所有路由都会放在这个表里
* `255` - local 表，保存本地接口地址、广播地址、NAT 地址，系统自己维护，不可修改

可以通过以下方式查看路由表和名字的对应关系，也可以通过命令自己定义路由表
```bash
$ cat /etc/iproute2/rt_tables
  255    local
  254    main
  253    default
  0      unspec
```

路由类型
* 主机路由 - 255.255.255.255 - UH - 指向单个 IP 地址的路由记录
* 网络路由 - 255.255.255.000 - UN - 代表主机可以到达的网络
* 默认路由 - 000.000.000.000 - UG - 当主机不能在路由表中查到目标主机时，数据包就发到默认路由上

**（二）路由常见操作**

* 查看路由表
```bash
$ route -n                   # route 命令查看路由表
$ ip route show              # ip-route 查看路由表
$ ip route show table local  # 查看 Local 路由表
```

* 添加主机路由
```bash
$ route add -host 10.0.0.10 gw 10.139.128.1 dev eth0
```
* 添加网络路由
```bash
$ route add -net 10.0.0.0 netmask 255.255.255.0 gw 10.139.128.1 dev eth0
$ route add -net 10.0.0.0/24 gw 10.139.128.1 dev eth0
$ route add -net 224.0.0.0 netmask 240.0.0.0 dev eth0
```
* 添加屏蔽路由
```bash
$ route add -net 224.0.0.0 netmask 240.0.0.0 reject
```
* 删除可用路由
```bash
$ route del -net 224.0.0.0 netmask 240.0.0.0
```
* 删除屏蔽路由
```bash
$ route del -net 224.0.0.0 netmask 240.0.0.0 reject
```
* 添加和删除默认网关
```bash
# 添加默认网关
$ route add default gw 192.168.1.1
# 删除默认网关
$ route del default gw 192.168.1.1
```
* 静态路由永久生效
```bash
$ route add -net 11.1.1.0 netmask 255.255.255.0 gw 11.1.1.1
# 保存在文件：/etc/sysconfig/static-routes 中，文件需要自己创建
any net 11.1.1.0 netmask 255.255.255.0 gw 11.1.1.1
```

**（三）策略路由实现多默认路由**

一般地说，在 Linux 系统路由表内只能有一条默认路由。当出站数据包根据目的 IP 地址选路失败后，会执行默认路由，交下一跳路由器（默认网关）转发数据包。现需要同时存在两条默认路由，数据包通过何种默认路由，由程序指定，数据包通过特定的路由规则转发到对应的路由器。
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

**（四）路由缓存**

在 3.6 以前的内核都包含带有垃圾收集器的 IPv4 路由选择缓存，删除的主要原因是，很容易对其发生 DOS 攻击，因为它会为每个流创建一个缓存条目，意味着可以随机发送数据包让其创建无数条缓存条目。请注意，在 IPv4 路由选择缓存的实现中，无论使用了多少个路由表，都只有一个缓存。

* Linux 查看路由缓存
```bash
$ ip route show cache 192.168.100.17
```


## 三、DPVS 路由

DPVS 是一个高性能的转发器，核心功能是负责转发 vip 流量到后端服务器。由于是基于 DPDK 框架开发的，网卡流量都由用户程序接管，无法直接使用 Linux 内核原生的网络子系统，DPVS 的路由转发功能完全是由自己来实现。下边从实际操作开始，一点点来了解 DPVS 路由模块。

**（一）dpvs 路由操作**

路由配置工具是使用 dpip 来进行相关的配置
```bash
# 查看路由信息
$ dpip route show

# 添加路由
$ dpip route add 192.168.100.0/24 dev bond0

# 删除路由
$ dpip route del 192.168.100.0/24 dev bond0

$ dpip route flush

# 查看 dpip route 帮助信息
$ dpip route help
```

**（二）路由配置的实现过程**

dpip 通过 unix socket 与 dpvs 的 master 线程进行通信，master 根据消息类型执行路由模块注册的 sockopt 函数：
```c
static struct dpvs_sockopts route_sockopts = {
    .version        = SOCKOPT_VERSION,
    .set_opt_min    = SOCKOPT_SET_ROUTE_ADD,
    .set_opt_max    = SOCKOPT_SET_ROUTE_FLUSH,
    .set            = route_sockopt_set,        // set
    .get_opt_min    = SOCKOPT_GET_ROUTE_SHOW,
    .get_opt_max    = SOCKOPT_GET_ROUTE_SHOW,
    .get            = route_sockopt_get,        // get
};
```
dpip 将用户输入的路由参数传递给 master，master 线程在当前 cpu 调用 `route_sockopt_set` 函数进行路由配置，设置完毕后通过 msg 将用户设置的路由信息通过多播的方式同步发送给各个 slave 线程，slave 线程在适当的时机执行路由处理函数 `route_msg_process` slave 线程也会按照传递的参数来设置相同的路由信息，所以 master 与各 slave 线程各自维护的路由信息完全一致。

> 1. 可以知道，dpvs 是为了避免路由锁操作，提高系统整体性能，每个 cpu 都维护了份路由信息，这样各个 cpu 读写时无需加锁。

**（三）路由的几种类型**

当我们执行 `dpip addr add xxx` 添加 IP 地址时，会同时添加相应的路由信息。除非添加 IP 时指定的掩码为 32，否则会同时添加一条 Local 路由和 Net 路由。

* Local 类型路由。local 类型路由的作用和 Linux 下的 local 路由表的功能基本一样，主要是记录本地的 IP 地址。我们知道进入的数据包，过了 prerouting 后是需要经过路由查找，如果确定是本地路由（本地 IP）就会进入 LocalIn 位置，否则丢弃或进入 Forward 代码位置了。Local 类型路由就是用来判定接收数据包是否是本机 IP，在 DPVS 调用的就是 `route4_input` 函数。
* Net 类型路由。数据包经过应用程序处理完毕后，要执行 output 时也需要根据目的 IP 来查找路由，确定从哪个网卡出去，下一跳等信息，然后执行后续操作。在 DPVS 调用的就是 `route4_output` 函数。

**（四）路由的相关数据结构字段**

local 类型路由采用是典型的哈希链表，每个 cpu 维护一张哈希表；而 net 路由却简单实用了普通的链表，猜想可能是 net 路由条目信息会比较少一些，性能影响不大。

```cpp
struct route_lcore {
    struct list_head local_route_table[LOCAL_ROUTE_TAB_SIZE];
    struct list_head net_route_table;
};
#define this_route_lcore        (RTE_PER_LCORE(route_lcore))
#define this_local_route_table  (this_route_lcore.local_route_table)
#define this_net_route_table    (this_route_lcore.net_route_table)
```
路由条目的数据结构字段也相对简单如下：
```cpp
struct route_entry {
    uint8_t netmask;
    short metric;
    uint32_t flag;
    unsigned long mtu;
    struct list_head list;
    struct in_addr dest;
    struct in_addr gw;
    struct in_addr src;
    struct netif_port *port;
    rte_atomic32_t refcnt;
};
```

DPVS 路由系统与邻居模块不同的时，路由信息完全是由管理员来配置，不存在动态学习的过程，理论上多核之间的路由信息同步设计上会稍简单了些。


