---
layout: post
date: "2020-01-10 18:51:32"
title:  "负载均衡 LVS 入门教程详解 - 连接表"
categories: 负载均衡
tags: lvs
author: 肖邦
---

* content
{:toc}

> 前面几篇文章分别是基本介绍、基础原理、操作实践几个方面来介绍了负载均衡 LVS 的一些知识。如果能够认真地把这几篇文章看完，理解并认真思考后应该会有所收获，至少能够掌握 LVS 的基础实现原理。




后边会继续从代码层面分析 lvs 的实现原理和细节。其实不只开发人员需要懂代码，运维人员如果能够深入到代码层面，相信对原理的理解会更加透彻，而且对平时运维中的 troubleshoting 肯定有帮助。如果感兴趣的话，可以一起来学习和探索 lvs 的具体实现原理。


## （一）连接表的概念作用

何为连接？我们知道 TCP 是面向连接的四层传输协议，一个 TCP 连接包括我们常说的五元组：src-IP、dst-IP、src-port、dst-port、protocol，也就是这 5 个元素来标识一个连接，能够代表唯一的会话。

lvs 是个四层负载均衡转发器，负责将数据包转给后端的服务器。那具体是怎么转发的呢，它有自己的调度方式，根据调度规则将数据包转发给正确的后端 Server。但是有个问题，每个数据包都需要根据特有的调度规则来进行调度转发的吗？

显然不是，因为每个 TCP 连接需要包含若干的数据包进行交互，如果每个 Packet 都按照调度规则来调度，比如按照 RR 模式来进行调度，那么该 TCP 连接相关的若干数据包则被轮询转发给了 N 个后端服务器，后端对这样的奇怪数据包是不感兴趣的（被无情丢弃），显然负载均衡需要将 TCP 连接的数据流的若干数据包转发至其中一台后端服务器上，而不是分散到多个服务器。

因此，lvs 若想正确转发数据包，需要维护记录数据包五元组代表的连接信息，包括首次建立连接时调度的 RS。那么后续的同一个连接（会话）的数据包到来后，查询系统维护的连接信息，查到该连接后就能确定这条连接对应的 RS 服务器，最终将数据包正确转发给服务器，这样就完成了将一个 TCP 连接转发给后端服务器了。


## （二）lvs 连接的基本结构

我们来看下 lvs 代码中是如何用数据结构来表示一条连接信息的，它定义的 struct 结构如下：

```cpp
struct ip_vs_conn {
	struct list_head        c_list;          /* 哈希链表头 */
	u16                      af;		     /* 协议族，代表 v4 或 v6 */
	union nf_inet_addr       caddr;          /* 客户端 IP 地址 */
	union nf_inet_addr       vaddr;          /* 客户端访问的 vip */
	union nf_inet_addr       daddr;          /* 后端 RS 服务器 IP 地址 */
	__be16                   cport;          /* 客户端请求源端口 */
	__be16                   vport;          /* 访问的业务端口，一般为 80 或 443 */
	__be16                   dport;          /* 后端服务器的端口，可以不同于 vport */
	__u16                   protocol;        /* 协议类型，如 TCP/UDP */
    atomic_t                refcnt;          /* 引用计数 */
	struct timer_list	timer;		         /* 连接对应的定时器 */
	volatile unsigned long	timeout;	     /* 超时时间 */
	spinlock_t              lock;            /* 状态转化时需要锁操作 */
	volatile __u16          flags;           /* 标记位 */
	volatile __u16          state;           /* 状态信息 */
	volatile __u16          old_state;       /* 保存上一个状态 */
	/* Control members */
	struct ip_vs_conn       *control;        /* Master control connection */
	atomic_t                n_control;       /* Number of controlled ones */
	struct ip_vs_dest       *dest;           /* 连接所调度的后端 RS */
	atomic_t                in_pkts;         /* 连接中进来数据包计数 */

	int (*packet_xmit)(struct sk_buff *skb, struct ip_vs_conn *cp,
                                       struct ip_vs_protocol *pp);

	struct ip_vs_app        *app;           /* bound ip_vs_app object */
	void                    *app_data;      /* Application private data */
	struct ip_vs_seq        in_seq;         /* incoming seq. struct */
	struct ip_vs_seq        out_seq;        /* outgoing seq. struct */
};
```
上述的结构体 `struct ip_vs_conn` 定义一个连接所需要多个字段元素，介绍下几个核心的字段：

* `c_list`：哈希表头，用来组织链表的关键字段。
* `caddr`：客户端 request 数据包的源 IP 地址。
* `vaddr`：客户端 request 数据包的目的 IP 地址，一般是业务对外提供访问的 IP，称为 vip
* `daddr`：连接新建时调度选择的一个后端 RS 服务器 IP 地址。
* `cport`：客户端 request 数据包的源 port
* `vport`：客户端 request 数据包的目标 port，一般业务端口为 80 或 443
* `dport`：连接对应的后端 RS 服务器端口，NAT 转发模式支持不同于 vport
* `protocol`：指的是四层传输协议，一般为 TCP 或 UDP
* `state`：连接是会记录状态的，根据数据包的交互而变化。
* `packet_xmit`：数据包发送的回调函数，不同的转发模式有不同的 xmit

lvs 能够提供高并发的转发能力，需要维护很多这样的连接信息，把这些连接信息组织起来就形成了 **`连接表`**。

## （三）连接表组织结构

lvs 连接表设计时出于各方面的考虑，比如查找性能等，采用了典型的哈希表结构将多个 `连接` 组织为连接表，组织结构如下图所示：

![ipvs hash table](/image/lb011ipvshashtable.png)

如图，根据 `caddr`、`cport`、`proto` 关键字段按照某种计算方法进行计算，得到一个哈希值。
```cpp
hash = ip_vs_conn_hashkey(cp->af, cp->protocol, &cp->caddr, cp->cport);
```
然后与连接表大小 `IP_VS_CONN_TAB_MASK` 值求和，得到在 `0-255` 范围内的一个整数，对应到图中的某个具体的槽位（桶），当前位置设置为链表头，然后将连接条目作为一个元素结点挂在此处的链表上。同理，其他连接信息会按照同样的方式被挂在如图所示的哈希链表上。

* 在需要查找连接表时，根据上述提到的相关字段计算出哈希值 hashkey，定位到具体的哈希桶位置，在该链表上遍历查找所要找的连接。
* 可以看出，哈希表的尺寸越大，哈希值冲突的可能性就越小，也就是挂在一个桶上的连接数据越少，这样在查找时也就越高效。
* 另外，虽然哈希表尺寸能够改善冲突的问题，不过设计出较好的哈希函数让连接能够均衡的分布在各个哈希桶里也很关键。

## （四）连接表的新建

何时应该新建连接呢，对于 TCP 协议来说，只有客户端发起的 `syn` 数据包时才能够触发新建连接，相同连接会话的其他数据包到来时，只需要查找存在的连接表即可继续后续的转发工作。
```cpp
	dest = svc->scheduler->schedule(svc, skb);
	if (dest == NULL) {
		return NULL;
	}
	/* 只有在成功调度可用的后端 RS 服务器后才去新建连接 */
	cp = ip_vs_conn_new(svc->af, iph.protocol,
			    &iph.saddr, pptr[0], &iph.daddr, pptr[1], &dest->addr,
			    dest->port ? dest->port : pptr[1], 0, dest);
```
新建连接的整个过程比较简单，大概分为几个步骤：
1. 分配 `连接` 所需要的内存资源空间
2. 初始化链表相关字段
3. 设置注册定时器工作函数
4. 填充核心关键字段
5. 关联绑定调度到的后端 RS 服务器
6. 初始化状态、超时时间、绑定 xmit 函数
7. 准备就绪，将 new_conn 挂在全局的哈希表上

到此新建连接完成，它的函数原型大概是这个样子：
```cpp
struct ip_vs_conn * ip_vs_conn_new(af, proto, caddr, cport, vaddr, vport, daddr, dport, flags, dest);
```

## （五）连接表的查找

新建连接后续的数据包到来是可以直接查找全局的连接表，找到该数据包所属的连接信息，也就能确定本次数据包需要转发给后端的哪一台 RS 服务器，通过相关操作后将数据包转发给连接所对应的 dest。

我们知道 DR 模式的话，只有客户端请求的流量会经过 LVS 服务器，当请求数据到达时，查找到支持的协议后，就会调用如下函数进行查找
```cpp
struct ip_vs_conn *ip_vs_conn_in_get(af, protocol, s_addr, s_port, d_addr, d_port);
```
通过函数参数也可以知道，是根据这 5 元组来进行查找的，步骤如下：
1. 根据关键字段调用 `ip_vs_conn_hashkey` 函数计算 hashkey
2. 加锁操作，因为连接表是全局的，在多 cpu 架构下，为了保证原子性操作。
3. 在对应的哈希桶上 `ip_vs_conn_tab[hash]` 遍历链表，如果和参数的各字段都相同，则 hit
4. 找到 hit 后，unlock 解锁，返回

如果是 NAT 模式的话，RS 的响应数据包也是会经过 LVS 服务器的，此时数据包的源 IP 是 RS 的 IP，目的 IP 是 VIP，因此需要使用另外一个函数来查找连接：
```cpp
struct ip_vs_conn *ip_vs_conn_out_get(af, protocol, s_addr, s_port, d_addr, d_port);
```
1. 首先，函数除了名称不同外，参数完全相同，不过这里要注意到这些源目的 IP 和端口与连接表的对应关系。
2. 计算 hashkey 时，是通过 d_addr 和 d_port，也就是客户端请求数据包的源 IP 和源端口，这样计算出的 hashkey 和新建连接的值是相等的。
3. 遍历 `ip_vs_conn_tab[hash]` 哈希桶上的链表，字段对比时区别 IP 和 Port 的方向。
```cpp
if (d_addr == cp->caddr &&      /* 反向数据：目的 IP 就是客户端 IP */
    s_addr == cp->daddr &&      /* 反向数据：源 IP 就是 RS 服务器 IP */
    d_port == cp->cport &&      /* 反向数据：目的端口是客户端 Port */
    s_port == cp->cport)        /* 反向数据：源端口是 RS 服务器 Port */
```

## （六）连接表的过期

上述关于连接的新建和查找比较容易理解，就是常见的添加与删除操作。而连接的删除比较特殊些，特殊在删除连接的时机上。
* 什么时候删除合适呢，我们不能简单地认为当连接彻底释放时删除就行。因为不是所有的连接在客户端和服务器间都能够正常的释放掉；如果某些连接没有正常释放，那么该连接就会长期一直残留在系统中，积累大量的无效连接，占用系统内存，影响系统性能。
* 怎么解决呢，在 lvs 系统中为每个连接维护了状态机，在不同阶段的状态设置不同时间的定时器，如果超过定时器时间，连接就会调用 `ip_vs_conn_expire` 自动释放。各阶段状态的超时时间设置如下：

```cpp
static int tcp_timeouts[IP_VS_TCP_S_LAST+1] = {
    [IP_VS_TCP_S_NONE]		    =	2*HZ,
    [IP_VS_TCP_S_ESTABLISHED]	=	15*60*HZ,
    [IP_VS_TCP_S_SYN_SENT]		=	2*60*HZ,
    [IP_VS_TCP_S_SYN_RECV]		=	1*60*HZ,
    [IP_VS_TCP_S_FIN_WAIT]		=	2*60*HZ,
    [IP_VS_TCP_S_TIME_WAIT]		=	2*60*HZ,
    [IP_VS_TCP_S_CLOSE]		    =	10*HZ,
    [IP_VS_TCP_S_CLOSE_WAIT]	=	60*HZ,
    [IP_VS_TCP_S_LAST_ACK]		=	30*HZ,
    [IP_VS_TCP_S_LISTEN]		=	2*60*HZ,
    [IP_VS_TCP_S_SYNACK]		=	120*HZ,
    [IP_VS_TCP_S_LAST]		    =	2*HZ,
};
```
上表可以看到，不同状态设置的超时时间不一样。比如 `ESTABLISHED` 状态设置时间比较长，可能你会担心如果 TCP 是长连接，输出传输要很长时间怎么办呢？其实是没有问题的，只要在 `ESTABLISHED` 状态超时时间范围内有数据包交互，就更会调用 `ip_vs_conn_put` 更新超时时间，也就是说在这个超时范围内只要有数据包（类似心跳）经过就不会中断释放掉连接。

```cpp
// 连接过期处理函数
static void ip_vs_conn_expire(unsigned long data);
```
1. 从哈希表中摘除该连接信息 `ip_vs_conn_unhash`
2. 删除定时器
3. 删除与 dest 绑定 `ip_vs_unbind_dest`
4. 释放内存资源

当然 lvs 也提供了立即释放连接的接口函数 `ip_vs_conn_expire_now` 直接将 timeout 设置为 0 ，相当于直接调用到了 `ip_vs_conn_expire` 函数。

## （七）连接表的锁

我们现在生产环境使用的服务器都是多核处理器，在 lvs 转发处理逻辑中需要频繁的查连接表，由于连接表是全局的一张表结构，绝大部分时间 N 个 cpu 会同时操作（包括查找、新建和删除）连接表，这样会导致连接表不一致的问题。因此，每个 cpu 对连接表进行相关操作时需要加上相应的 ReadLock 或 WriteLock，才能保证连接表的原子性操作。

* **全局锁操作**。假如我们为连接表定义一个全局锁，每个 cpu 在对连接表做读写操作时，都需要先加锁，获取锁后进行相关操作，操作完毕再释放锁资源。这样在高并发的业务场景下，同一时刻锁只能被一个 cpu 占用，因此各 cpu 之间对锁的竞争会非常严重，导致整体 lvs 性能会随着 cpu 个数增加而逐渐下降。
* **局部锁优化**。lvs 在设计上没有使用全局的锁，而是根据连接表的哈希结构，为 N 个哈希桶设置一个 Lock，这里的 N 为 256，也就是 256 个桶使用一个锁，总共有 4096 个哈希桶。如果有 16 个 cpu 的话，那么同一时间内平均每个 cpu 都可能占有一个锁，从系统整体上看，cpu 对锁操作的竞争就会大大减弱，提高了系统整体的转发能力。
  ```cpp
  static inline void ct_read_lock(unsigned key)
  {
      read_lock(&__ip_vs_conntbl_lock_array[key&CT_LOCKARRAY_MASK].l);
  }
  // CT_LOCKARRAY_MASK = 16
  ```
* **为每个桶绑定一个锁，进一步优化**。局部优化能够很大程度上提高系统性能，同样思路，系统资源允许的情况下，我们可以为每个哈希桶分配设置一个 "锁"，那么各 cpu 之间锁竞争就会变得更小啦，系统性能肯定会有更好地优化。
* **无锁化，每个 cpu 一张连接表**。既然每个 cpu 都要访问连接表，且会引起锁的竞争恶化，为何不给每个 cpu 维护一张连接表，每个 cpu 只维护自己相关数据连接组成的连接表，每个 cpu 读写 连接表时互不干扰，这样就实现无锁化，完美的避开了锁。
  - 很多公司维护的私有版本 lvs 都做过类似方面的优化，每个 cpu 一张连接表
  - 对于 DR 模式来说，能够较容易实现，因为只有入的流量，网卡按照 4 元组哈希，同一个数据流肯定会打到同一个 cpu 上去
  - 对于 NAT 模式来说，就比较麻烦不好处理，因为 NAT 回包还会经过 lvs，回包到达 lvs 网卡时，按照 4 元哈希后，数据包可能到达网卡队列 2 了，而近来的请求数据包到达网卡队列 1 了，那么同一个连接的数据包不在同个 cpu 上就会出现异常现象了，因为回包到达的 cpu 无法处理该数据包（没有连接信息）。

## （八）连接表可以如何优化

对于上边的内容是有些可以优化的空间的：
* 连接表哈希桶数量可以增加，如果内存空间允许的话（一般内存不是啥大问题），这样可以减少哈希冲突，让每个哈希桶中挂载的数据尽可能的少，这样在查找遍历链表时效率会大大提高。
* 连接表的老化时间，每个状态阶段都有设置固定时间的老化时间，我们可以适当缩小 `IP_VS_TCP_S_ESTABLISHED` 等状态的超时时间，这样能够让连接表老化的更快，这样能够支持的并发连接数也能够有所提高。
* 连接表关于锁的优化，即使不做成 per-cpu 方式的连接表，也可以适当优化，让每个哈希桶设置单独的锁，这样一定程度上能够减少锁竞争带来的性能损耗。

上述几点也是一般程序优化经常使用的方式，也在想这么个问题，为何处理内核态的 lvs 没有再进一步做优化呢？猜想可能是这样的，Linux 本身是一个非常通用的操作系统，面向的是大众用户，需要支持运行各种服务，而 lvs 只是 Linux 冰山一角的功能，如果优化程度太高，会占用系统过多的资源，这可能不是 Linux 所希望的，不过内核原版 lvs 性能也能够满足大部分中小企业的需求。

> 简单总结，连接表的核心功能就是起到 `连接追踪`，包括两方面内容：连接保持和连接老化。

