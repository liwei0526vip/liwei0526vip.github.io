---
layout: post
date: "2020-02-09 20:23:10"
title:  "高性能负载均衡 DPVS 邻居子系统的实现"
categories: 负载均衡
tags: lvs dpvs
author: 肖邦
---

* content
{:toc}
<h1><center>DPVS 邻居模块（neigh）</center></h1>

> 数据包从服务器转发的"下一跳"，一般情况下是网关（路由器）或者是同网段目标服务器，数据包需要将 DST MAC 地址设置为 "下一跳" 的网卡 MAC 地址。系统需要维护一张邻居表，记录下一跳目标服务器的 MAC 地址，邻居一般是动态学习的，根据需要通过 ARP 数据包交互过程中学到的，邻居条目需要有定时器记录超时时间。也可以通过手工配置的方式，不过这样的邻居项是静态的，不需要超时时间，在系统调测是很有用。




IPv4 的邻居是通过 ARP 协议来实现的，ARP 是一个链路层地址解析协议，它以 IP 为值，查询持有该 IP 地址主机的 MAC 地址。IPv6 是通过 ICMPv6 的 NS NA 类型消息进行交互实现邻居发现的。基本实现都是请求与响应的方式。

在 DPVS 系统中，开启的是多 CPU 线程模型，为了避免锁操作，为每个 CPU 维护了一张邻居表，各个 CPU 之间的邻居表是冗余的，同步的。

## neigh状态机

邻居学习过程的核心内容主要是邻居表项的各种状态机的变化，DPVS 的 neigh 状态机与 Linux 系统相比，作了一些简化，如图：

![image](https://raw.githubusercontent.com/liwei0526vip/blogimg/master/lb009dpvsneigh.png)

* `NONE`：最初始的状态
* `SEND`：已经发送了 ARP 请求，还未收到 ARP 响应，和 Linux 上看到的 Incomplete 类似
* `REACHABLE`：表示邻居可达的状态，有效邻居缓存。
* `PROBE`：当 `reachable` 状态超时后进入 `probe` 而不是 `none`，因为可达状态超时后，我们总期望邻居仍然是可达的，再次发送探测 ARP 请求，这主要是为了性能优化考虑的。
* `DELAY`：由 `probe` 发出 arp 请求后变为 `delay` 状态，在超时前收到 arp 响应就会转为 `reachable` 可达状态。

```cpp
static struct nud_state nud_states[] = {
/*                sNNO, sNSD, sNRE, sNPR, sNDE*/
/*send arp*/    {{sNSD, sNSD, sNKP, sNDE, sNDE}},
/*recv arp*/    {{sNRE, sNRE, sNRE, sNRE, sNRE}},
/*ack confirm*/ {{sNKP, sNKP, sNRE, sNRE, sNRE}},
/*mbuf ref*/    {{sNKP, sNKP, sNKP, sNPR, sNKP}},
/*timeout*/     {{sNNO, sNNO, sNPR, sNNO, sNNO}},
};
```
arp 报文格式如下：
```
+----------+----------+-----+-----+----------+------------------------------+
| 硬件类型 | 协议类型 |hwlen|plen |    op    |          发送者硬件地址      |
+----------+----------+-----------+----------+-----------+------------------+
|   发送者IP地址      |          目标硬件地址            |    目标IP地址    |
+---------------------+----------------------------------+------------------+
```

邻居表主要是由动态学习到的，DPVS 也支持从命令行控制层面配置静态的邻居条目。

## 邻居的控制面

通过命令行 `dpip neigh` 方式可以配置静态的邻居缓存，静态邻居没有超时时间，dpip 工具看到的条目后边带有 `static` 关键标记。

```bash
$ dpip neigh show
ip: 192.168.11.15 mac: f8:f2:1e:21:19:e0 state: REACHABLE dev: bond0  core: 1  "static"
ip: 192.168.11.15 mac: f8:f2:1e:21:19:e0 state: REACHABLE dev: bond0  core: 2  "static"
ip: 192.168.11.15 mac: f8:f2:1e:21:19:e0 state: REACHABLE dev: bond0  core: 3  "static"
ip: 192.168.11.15 mac: f8:f2:1e:21:19:e0 state: REACHABLE dev: bond0  core: 4  "static"
```
邻居注册的控制面接口如下：
```c
static struct dpvs_sockopts neigh_sockopts = {
    .version     = SOCKOPT_VERSION,
    .get_opt_min = SOCKOPT_GET_NEIGH_SHOW,
    .get_opt_max = SOCKOPT_GET_NEIGH_SHOW,
    .get         = neigh_sockopt_get,  // get 接口

    .set_opt_min = SOCKOPT_SET_NEIGH_ADD,
    .set_opt_max = SOCKOPT_SET_NEIGH_DEL,
    .set         = neigh_sockopt_set,  // set 接口
};
```
* `neigh_sockopt_get`：用于获取系统当前的邻居表信息，代码执行在控制面线程，master 线程会给各个 worker 线程发送 `MSG_TYPE_NEIGH_GET` 消息，worker 线程分别会将邻居信息通过队列发送给 master，master 线程收到所有 worker 邻居消息汇总并拷贝发送给 dpip 命令行。
* `neigh_sockopt_set`：接口使用来设置静态邻居，可以 add 和 del 操作。代码执行在控制面线程，master 会根据添加或删除的信息，将要操作的信息结构通过 neigh_ring[coreid] 队列发送给每个 worker 线程，worker 线程中通过调用注册的 `NETIF_LCORE_JOB_SLOW` 类型的 Job（neigh_process_ring）来执行邻居条目的增加或删除，以及邻居状态的转换。

## 动态学习邻居

> 代码路径 >>> `"src/neigh.c"` DPVS 邻居模块。

在上层协议处理完毕，并确定下一跳后，就需要确定二层链路信息。系统会从当前 cpu 对应的邻居表中查找 "下一跳" 对应邻居条目，如果查到的邻居缓存是有效的，则会调用 `neigh_fill_mac` 函数进行 MAC 信息填充，填充完毕后就可以调用 netif_xmit 向网卡发送数据包了。如果没有找到，或者查找到的邻居状态无效，则需要根据实际状态进行邻居学习了。

* **响应邻居 ARP 请求**。如果有同网段的服务器或者路由器网关，需要解析 DPVS 相关 IP 的 MAC 地址，就会向 DPVS 发送 ARP 请求，DPVS 确认目标 IP 是本机 IP 后，就将 IP 对应的 MAC 地址作为响应 ARP 包的 target mac 字段，发送给请求的邻居，这样就完成了 ARP 响应的过程。
* **邻居学习过程**。当 dpvs 确定下一跳后准备发送数据包时，如果没有缓存该邻居条目，此时就需要去学习下一跳的 mac 硬件地址了。dpvs 会构建并主动发送 arp 请求报文，等相应的 arp 响应报文到达之后，就获得了有效的邻居缓存记录，将缓存的 mac 地址填充报文并发送出去。

对于 arp 报文，有专门的报文类型处理函数 `neigh_resolve_input`，是所有 arp 报文的处理入口函数。
* 函数首先会检查 target ip 是否为 dpvs 所属的 IP，如果不是则退出函数。
* 如果 arp 操作类型是 **`ARP_OP_REQUEST`**，则是直接读取 IP 所在端口的硬件地址，将对应的 mac 地址作为 reply 报文的 target mac 字段，发送给请求 neigh
* 如果 arp 操作类型是 **`ARP_OP_REPLY`**，则会创建或修改邻居条目，将邻居条目的状态机转换为 `reachable`。这时不管当前是什么状态，只要收到 arp reply 报文，都会更新为可达状态。
* **关于 cpu 之间邻居的同步**。在每个 cpu 收包阶段的前期，如果收到 arp 响应报文，就会将相应的 mbuf 克隆多份，向每个 cpu arp_ring 队列发送此报文，其他 cpu 线程在执行逻辑中会调用 `lcore_process_arp_ring` 进行处理，也会调用 `lcore_process_packets` 函数进行正常包的处理流程。这样就实现了各 cpu 之间邻居表的同步。
* **邻居的超时**。邻居的每个状态都有超时时间，在到达超时后会执行相应的超时事件函数 `neigh_entry_expire`，会取消定时器；摘除哈希表；将邻居上对应的待处理数据包 mbuf 释放掉。


## 问题思考

* 在 dpvs 数据包要转发时，对应的邻居缓存无效或不存在，dpvs 需要阻塞等待吗？
> dpvs 在数据包需要转发而邻居信息尚未准备好时，待发送业务数据包并不会阻塞在这里，而是将 mbuf 数据包加入到该邻居的 `neigh_mbuf_list` 队列中去，转换邻居相应的状态，继续执行别的代码逻辑。待 cpu 收到该邻居的 reply 数据包后，就会更改邻居的状态信息，同时会将邻居 `neigh_mbuf_list` 队列中缓存的 mbuf 全部发送出去，这么做能够避免阻塞等待，极大提高系统的处理性能。

* 关于 confirm 邻居作用的理解。
> 其实 neigh 的状态可以再简化，不要 probe 和 delay 状态，这样 neigh 超时后就直接释放。这样做理论上可以，不过大多数情况下邻居变化的概率较小，这时邻居绝大多数情况下是没有变化的，如果直接将可达状态完全释放掉，会严重影响效率。如果邻居的状态在 probe 或 delay 时，上层协议有相关数据包的传输确认，这时我们可以认为邻居是可用的，仍然转变为 `reachable` 的有效状态，减少了邻居的再次请求与响应的过程，有效提高了性能。在 dpvs 里相关的函数是 `neigh_confirm` 在收到 ack 并条件满足时，会触发一次 confirm 动作。

## IPv6 邻居发现

IPv6 的邻居发现相关操作进行了优化，没有使用 ARP 广播的方式来执行邻居请求，而是统一使用 ICMPv6 协议消息来实现邻居发现和路由发现的过程。

DPVS 中只是简单实现了邻居请求和邻居通告的功能，暂不支持路由请求、路由发现、重定向等功能。从原理上来看，实现方式也是通过请求与响应的交互进行的。

> 代码路径 >>> `"src/ipv6/ndisc.c"` IPv6 邻居发现模块，该模块涉及较多 IPv6 相关的知识点，就不整理细节知识了。

DPVS 邻居发现入口函数是 `ndisc_rcv` ，根据 icmp6_type 来区分：
* `ND_NEIGHBOR_SOLICIT` 类型调用 `ndisc_recv_ns`，主要针对接收到的邻居请求数据包的逻辑处理，包括状态转换与多核邻居信息的同步、mbuf_cache 的处理、发送 NA 响应消息，另外也包含了 DAD 地址冲突检测的逻辑代码。
* `ND_NEIGHBOR_ADVERT` 类型调用 `ndisc_recv_na`，主要是针对接收到的邻居通告数据包的逻辑处理，包括状态机转换，多核邻居信息同步、mbuf_cache 处理等。

注：和 IPv4 邻居不同的是，接收到对方邻居响应信息 NA 时，同时也学习了对方的 MAC 地址，这样可能会更有效率一些。

