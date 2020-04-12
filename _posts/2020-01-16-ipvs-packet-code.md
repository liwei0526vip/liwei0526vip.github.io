---
layout: post
date: "2020-01-16 19:00:33"
title:  "负载均衡 LVS 入门教程详解 - 主流程代码分析"
categories: 负载均衡
tags: lvs
author: 肖邦
---

* content
{:toc}
> 前边系统介绍了 lvs 的两个核心概念**连接表**和**调度算法**，它们是 lvs 的核心内容，是其实现的原理基础。




连接表的本质其实就是连接跟踪，维护保持一个连接相关的数据包的关联性；调度算法是 lvs 实现后端服务器调度的依据原则。理解了这两个概念后，再来分析 lvs 具体实现的过程就会比较容易了，本文会从代码层面来简述 lvs 各种模式的实现原理，理解本文后再回头去看基础原理那篇文章应该会有知其所以然的感觉。

## 一、关于 netfileter 内容

我们知道 lvs 是基于 netfilter 框架的，netfilter 整体上还是比较复杂的，它支持数据包在网络栈传输路径各个地方（挂载点）注册回调函数，从而对数据包执行各种操作，如修改地址或端口、丢弃数据包等。

![netfilter](https://www.linuxblogs.cn/image/lb003netfilter.png)

如图中，在网络栈中有 5 个地方设置了 netfilter 挂载点：
* `NF_INET_PRE_ROUTING` - 这是所有入栈数据包的第一个挂载点，它处于路由选择子系统查找之前，挂载点位于 `ip_rcv` 中，v6 是 ipv6_rcv。
* `NF_INET_LOCAL_IN` - 对于所有发送给当前主机的入栈数据包，经过挂载点 NF_INET_PRE_ROUTING 并执行路由选择子系统查找后，都将到达这个挂载点。挂载点位于方法 `ip_local_deliver` 中，v6 对应函数是 ip6_input 中。
* `NF_INET_FORWARD` - 对于所有要转发的数据包，经过挂载点 NF_INET_PRE_ROUTING 并执行路由选择子系统查找后，都将到达这个点。挂载点位于 `ip_forward` 中，而 v6 对应函数是 ip6_forward。
* `NF_INET_POST_ROUTING` - 所有要转发的数据包和当前主机生成的数据包都要经过此挂载点，IPv4 中这个挂载点位于方法 ip_output 中，IPv6 位于方法 ip6_finish_output2 中。
* `NF_INET_LOCAL_OUT` - 当前主机生成的所有出站数据包都在经过这个挂载点，IPv4 中挂载点位于方法 __ip_local_out 中，而在 IPv6 中位于方法 __ip6_local_out 中。

lvs 在不同的模式（DR、NAT、Tunnel）中，根据不同的需求在 netfilter 中注册了多个挂载点，在每个挂在点上执行相应的代码程序，从而实现相应的功能。

```cpp
static struct nf_hook_ops ip_vs_ops[] __read_mostly = {
	{
		.hook		= ip_vs_in,
		.owner		= THIS_MODULE,
		.pf		    = PF_INET,
		.hooknum    = NF_INET_LOCAL_IN,
		.priority   = 100,
	},
	{
		.hook	    = ip_vs_out,
		.owner		= THIS_MODULE,
		.pf		    = PF_INET,
		.hooknum    = NF_INET_FORWARD,
    	.priority   = 100,
	},
	{
		.hook		= ip_vs_forward_icmp,
		.owner		= THIS_MODULE,
		.pf		    = PF_INET,
		.hooknum    = NF_INET_FORWARD,
		.priority   = 99,
	},
	{
		.hook		= ip_vs_post_routing,
		.owner		= THIS_MODULE,
		.pf	        = PF_INET,
		.hooknum    = NF_INET_POST_ROUTING,
		.priority   = NF_IP_PRI_NAT_SRC-1,
	},
};
```
这是 lvs 在 IPv4 实现中注册的几个钩子函数，也是相应的入口位置，下边内容会分别从 DR、NAT、Tunnel 不同模式来总结数据包从入口，主逻辑处理，输出发送等过程。


## 二、DR 模式的数据包处理流程

首先，DR 模式使用到的 hook 函数是 `ip_vs_in`，它是 DR 模式时数据包的入口位置。从上边注册代码段中，可以知道它注册的挂载点位置是 `NF_INET_LOCAL_IN`，也就是说数据包到目前位置时，已经通过 prerouting 和路由系统查找。我们现在假设某个客户端访问 vip 是 182.120.11.20 端口为 80 的数据包，将会经过如下步骤，来实现 DR 模式的数据包转发功能。

**（一）查找四层协议**

通过 IP 报文中的 protocol 字段，查找并确定该协议是否被 lvs 所支持。lvs 能够支持 tcp、udp、ah、esp 协议，不过本文重点以最常用的 tcp 协议来介绍。
```cpp
pp = ip_vs_proto_get(iph.protocol);
if (unlikely(!pp))
	return NF_ACCEPT;
```
调用上述函数查找该数据包四层协议是否在已经注册的协议类型列表中，如果不是支持的协议，也就没有必要再继续执行后续流程，就直接退出 lvs 代码主流程，由后续内核处理流程来执行，可能是内核其他应用程序感兴趣的数据包，返回 NF_ACCEPT。

**（二）调度并新建连接信息**

接着，调用连接表查找函数 `conn_in_get` 
```cpp
cp = pp->conn_in_get(af, skb, pp, &iph, iph.len, 0);
```
根据数据包 5 元组相关字段进行查找，这里假设是一个新连接的请求数据包，那么在连接表中是查不到连接表项的，因此这时 cp = NULL，然后会调用 `conn_schedule` 函数
```cpp
pp->conn_schedule(af, skb, pp, &v, &cp)
```
这个函数主要是调度一个新的 RS ，然后新建立一个连接条目：
* 我们主要以 tcp 来说明，所以这里使用的调度函数是 `tcp_conn_schedule`
* 首先要说明，并不是说有数据包来到这里都会执行调度流程，需要符合 2 个前提条件
  - 数据包必须携带 SYN 标记
  - 数据包所访问请求必须是我们定义的 service 才行，也就是能够正常获取到 svc
     ```cpp
     svc = ip_vs_service_get(af, skb->mark, iph.protocol, &iph.daddr, th->dest)
     ```
* 满足上述条件后，执行调度函数 `ip_vs_schedule`
  ```cpp
  dest = svc->scheduler->schedule(svc, skb);
  ```
  调度函数主要是负责从 svc 所指向的 dests 中根据调度算法选择出一个合适的后端服务器（dest）
* 在成功调度 dest 后，调用 `ip_vs_conn_new` 函数来创建真正的连接 Entry
  - 分配 cp 内存资源
  - 设置定时器
  - 填充相关字段
  - 绑定并设置 dest 相关信息，调用 `ip_vs_bind_dest` 函数
  - 绑定 conn 所需的 xmit 函数，调用 `ip_vs_bind_xmit` 函数
  - 将 cp 连接条目 hash 到全局的连接表中
* 至此，连接相关的信息已经准备完毕了。
> 注1：如果客户端第 2 个数据包到来时，就不需要再次调度 RS 和新建连接表项了，直接根据查找到的 cp 来执行下列的相关流程。
>
> 注2：我们知道 DR 模式数据包从服务器响应之后是直接从 Server 发给 Client 的，无需经过 lvs，所以到达 lvs 的所有数据包都是客户端发过来的。

**（三）更新设置连接的状态机**

还记得我们在「连接篇」文章中说过，对于一个连接来说，尤其处于每个 tcp 状态时的连接对应的过期时间设置的都不相同，因此我们需要记录连接的各种状态的转换。另外如果我们需要统计活跃连接和非活跃连接的话，也必须采用当前连接所处的状态为依据。

下边我们对着代码来看 tcp 的状态机设置函数 `tcp_state_transition` 是如何进行更新状态机的。其实主要的核心函数是 `set_tcp_state`
```cpp
static inline void
set_tcp_state(struct ip_vs_protocol *pp, struct ip_vs_conn *cp,
	      int direction, struct tcphdr *th)
{
	int state_idx;
	int new_state = IP_VS_TCP_S_CLOSE;
	int state_off = tcp_state_off[direction];

	if (cp->flags & IP_VS_CONN_F_NOOUTPUT) {
		if (state_off == TCP_DIR_OUTPUT)
			cp->flags &= ~IP_VS_CONN_F_NOOUTPUT;
		else
			state_off = TCP_DIR_INPUT_ONLY;
	}

	if ((state_idx = tcp_state_idx(th)) < 0) {
		IP_VS_DBG(8, "tcp_state_idx=%d!!!\n", state_idx);
		goto tcp_state_out;
	}

	new_state = tcp_state_table[state_off+state_idx].next_state[cp->state];

tcp_state_out:
	if (new_state != cp->state) {
		struct ip_vs_dest *dest = cp->dest;
		if (dest) {
			if (!(cp->flags & IP_VS_CONN_F_INACTIVE) &&
			    (new_state != IP_VS_TCP_S_ESTABLISHED)) {
				atomic_dec(&dest->activeconns);
				atomic_inc(&dest->inactconns);
				cp->flags |= IP_VS_CONN_F_INACTIVE;
			} else if ((cp->flags & IP_VS_CONN_F_INACTIVE) &&
				   (new_state == IP_VS_TCP_S_ESTABLISHED)) {
				atomic_inc(&dest->activeconns);
				atomic_dec(&dest->inactconns);
				cp->flags &= ~IP_VS_CONN_F_INACTIVE;
			}
		}
	}

	cp->timeout = pp->timeout_table[cp->state = new_state];
}
```
从上述代码上来看，该函数主要做了这几件事情：
1. 根据数据包 tcp 标记字段来更新当前状态机
2. 更新连接对应的统计数据，包括：活跃连接和非活跃连接
3. 根据连接状态，设置超时时间

下面我们来具体分析下状态机状态转换过程，是通过索引从 `tcp_state_table` 数组中获取到下一个状态：
```cpp
new_state = tcp_state_table[state_off+state_idx].next_state[cp->state];
// @new_state：是查找得到的最新状态
// @cp->state：是查找前conn所处的当前状态
```
这个数组的具体内容大概如下：
```cpp
static struct tcp_states_t tcp_states [] = {
/*	INPUT */
/*        sNO, sES, sSS, sSR, sFW, sTW, sCL, sCW, sLA, sLI, sSA	*/
/*syn*/ {{sSR, sES, sES, sSR, sSR, sSR, sSR, sSR, sSR, sSR, sSR }},
/*fin*/ {{sCL, sCW, sSS, sTW, sTW, sTW, sCL, sCW, sLA, sLI, sTW }},
/*ack*/ {{sCL, sES, sSS, sES, sFW, sTW, sCL, sCW, sCL, sLI, sES }},
/*rst*/ {{sCL, sCL, sCL, sSR, sCL, sCL, sCL, sCL, sLA, sLI, sSR }},

/*	OUTPUT */
/*        sNO, sES, sSS, sSR, sFW, sTW, sCL, sCW, sLA, sLI, sSA	*/
/*syn*/ {{sSS, sES, sSS, sSR, sSS, sSS, sSS, sSS, sSS, sLI, sSR }},
/*fin*/ {{sTW, sFW, sSS, sTW, sFW, sTW, sCL, sTW, sLA, sLI, sTW }},
/*ack*/ {{sES, sES, sSS, sES, sFW, sTW, sCL, sCW, sLA, sES, sES }},
/*rst*/ {{sCL, sCL, sSS, sCL, sCL, sTW, sCL, sCL, sCL, sCL, sCL }},

/*	INPUT-ONLY */
/*        sNO, sES, sSS, sSR, sFW, sTW, sCL, sCW, sLA, sLI, sSA	*/
/*syn*/ {{sSR, sES, sES, sSR, sSR, sSR, sSR, sSR, sSR, sSR, sSR }},
/*fin*/ {{sCL, sFW, sSS, sTW, sFW, sTW, sCL, sCW, sLA, sLI, sTW }},
/*ack*/ {{sCL, sES, sSS, sES, sFW, sTW, sCL, sCW, sCL, sLI, sES }},
/*rst*/ {{sCL, sCL, sCL, sSR, sCL, sCL, sCL, sCL, sLA, sLI, sCL }},
};
```
上述表看着很复杂，我们先来搞懂 state_off 和 state_idx 两个变量的具体作用：
* **state_off**。在 DR 模式下，取值为 TCP_DIR_INPUT_ONLY
* **state_idx**。是根据数据包 tcp 头部标记位来确定的，比如 syn=0，fin=1，ack=2，rst=3。也只有这些标记能够改变 tcp 的连接状态
* **cp->state**。是当前 tcp 连接所处于的状态。

1. 假设这个数据包是携带 syn 标记，那么数组索引到的就是 INPUT-ONLY 组中 syn
```cpp
/*syn*/ {{sSR, sES, sES, sSR, sSR, sSR, sSR, sSR, sSR, sSR, sSR }},
```
因为 cp->state 初始值为 0，因此最终得到的 new_state = sSR，也就是 `IP_VS_TCP_S_SYN_RECV` 状态了
2. 假设这个数据包是携带 ack 标记，那么数组索引到的就是 INPUT-ONLY 组中的 ack
```cpp
/*ack*/ {{sCL, sES, sSS, sES, sFW, sTW, sCL, sCW, sCL, sLI, sES }},
```
因为 cp->state 值为 sSR，因此最终得到的 new_state = sES，也就是 `IP_VS_TCP_S_ESTABLISHED` 状态，对于 lvs 来说三次握手的 ack 到来意味着连接的成功建立，进入 est 状态。

关于状态机的转换就不逐一分析了，需要注意的是 DR 模式下与我们平时所理解的状态有所不同，因为后端服务器响应数据包是直接发给 client 的不经过 lvs，所以看到上边数据分类区分 INPUT、OUTPUT、INPUT-ONLY

**（四）转发数据包**

DR 的基本处理逻辑已经完毕了，到了数据包转发阶段，会调用相应的 xmit 函数
```cpp
ret = cp->packet_xmit(skb, cp, pp);
// 对于 DR 模式来说，调用的其实是 ip_vs_dr_xmit
int ip_vs_dr_xmit(struct sk_buff *skb, struct ip_vs_conn *cp, struct ip_vs_protocol *pp)
```

* 路由查找。以 dest 地址为路由的 daddr 进行查找，调用 `ip_route_output_key` 进行路由查找，由于 DR 后端的 dest 都是同网段的，因此路由查找的下一跳直接就是 dest 服务器，并将此路由信息设置到 dst_entry。
* mtu 检查。检查当前数据包的大小与路由相关 mtu 的关系，如果大于路由 mtu 且 IP 头部设置禁止分片的选项，那么 lvs 只能丢弃该数据包，调用内核函数 `icmp_send` 发送 `ICMP_DEST_UNREACH` 数据包给源客户端上报此错误信息。
* 为 skb 设置路由。将查找到的路由 dst_entry 信息绑定到 skb 结构上，以供后续流程使用。
* 转发数据包。设置路由后，就可以转发数据包了，使用 `IP_VS_XMIT` 转发，本质上就是调用 NF_HOOK 函数来执行 LOCAL_OUT 挂载点相关的代码。
```cpp
NF_HOOK(pf, NF_INET_LOCAL_OUT, (skb), NULL,	(rt)->u.dst.dev, dst_output);
```
上述代码会执行 `LOCAL_OUT` 挂载点的代码，以及相关内核后续正常的流程，相关大概流程如下：
```
local_out -> ip_output() -> ip_finish_output -> ip_finish_output2 -> neigh_hh_output
```
其实可以简单理解为，此时将 dest 设置为路由目标地址后，直接将数据包送往到 output 链上，直接执行后续相关的流程，包括邻居子系统执行的 mac 地址替换阶段，最终数据包转发出去后，目标 mac 地址就是某个 dest 的 mac 地址，也就实现了 mac 地址的 "修改"，最终完成了 DR 模式的数据包转发。

> 注1：我们在基础原理篇讲 DR 模式 lvs 会替换 mac 地址，其实严格从代码上分析，lvs 代码本身没有修改 mac 地址，而是借助于内核数据包的处理流程，在二层邻居系统帮助修改的而已。
>
> 注2：如果想要非常清晰的理解最后的 xmit 过程，需要研究内核协议栈数据包的流程细节了。


## 三、NAT 模式的数据包处理流程

接下来分析下 NAT 模式下的数据包处理流程，NAT 模式和 DR 模式区别还是比较大的，NAT 响应数据包也会经过 lvs 设备，且回包经过 lvs 时的目的 IP 是客户端 IP 地址，在 `LOCAL_IN` 挂载点是截获不了 NAT 响应数据包的，因此 NAT 需要注册两个挂载点：
* `ip_vs_in` - 这个 hook 函数主要是用来处理从客户端到服务器方向的数据包流程，比较像 DR 模式的流程。
* `ip_vs_out` - 这个 hook 函数主要是用来处理 NAT 响应数据包，也就是从服务器发给客户端的包，因为它目的 IP 不是 lvs 本地 IP，在 `local_in` 挂载点无法截获，需要通过注册在 `forward` 挂载点的 `ip_vs_out` 函数来进行处理。

**（一）NAT 请求数据包处理流程**

NAT 模式下，客户端的请求数据包都会经过 `ip_vs_in` 函数进行一些列的逻辑处理，这些处理从步骤上和 DR 模式非常相似，所以这里就会省略或简化一些和 DR 类似的一些内容。

0. NAT 请求数据包入口函数是 `ip_vs_in`
1. 查找四层协议，确认请求数据包是 lvs 支持的协议，不支持的当然不会处理了
2. 查找相关 svc，调度新的 RS 服务器，然后新建连接表项，hash 连接表
3. 更新设置连接相关的状态机，只不过有一些区别是，NAT 状态机表分组选择的是 INPUT，有兴趣的话可以对照着仔细分析各状态的变化。
```cpp
/*	INPUT */
/*        sNO, sES, sSS, sSR, sFW, sTW, sCL, sCW, sLA, sLI, sSA	*/
/*syn*/ {{sSR, sES, sES, sSR, sSR, sSR, sSR, sSR, sSR, sSR, sSR }},
/*fin*/ {{sCL, sCW, sSS, sTW, sTW, sTW, sCL, sCW, sLA, sLI, sTW }},
/*ack*/ {{sCL, sES, sSS, sES, sFW, sTW, sCL, sCW, sCL, sLI, sES }},
/*rst*/ {{sCL, sCL, sCL, sSR, sCL, sCL, sCL, sCL, sLA, sLI, sSR }},
```
4. NAT 数据包转发 xmit 阶段，它对应的 xmit 函数是 `ip_vs_nat_xmit`
   - 路由查找。根据 dest 为 daddr 进行路由查找
   - 检查 mtu。 检查 mtu 与包大小，如果包大于 mtu，且不允许分片的话，就会向客户端发送 "ICMP_DEST_UNREACH" 错误数据包，并丢弃该请求数据包。
   - 修改数据包四层信息。NAT 比 DR 模式多了四层处理的逻辑，因为 NAT 支持修改目的端口，因此会执行 `dnat_handler` 操作，对应的实际函数 `tcp_dnat_handler`。
     ```cpp
     /* 1. 修改目的端口为 dport */
     tcph->dest = cp->dport;
     /* 2. 重新计算 tcp 校验和，计算前需要将 check 字段置为 0 */
     tcph->check = 0;
	 tcph->check = csum_tcpudp_magic(cp->caddr.ip, cp->daddr.ip, skb->len - tcphoff, cp->protocol, skb->csum);
     ```
   - 修改数据包三层信息。NAT 需要修改目标 IP 地址为 RS 服务器 IP 地址，否则 RS 服务器不会接受此数据包（非本机 IP 默认会丢弃）。
     ```cpp
     /* 1. 修改目的 IP 地址为 daddr */
     ip_hdr(skb)->daddr = cp->daddr.ip;
     /* 2. 修改 IP 后，需要重新计算 3 层校验和 */
	 ip_send_check(ip_hdr(skb));
     ```
   - 转发数据包。设置路由后，并修改了数据包的目的 IP，调用 `IP_VS_XMIT` 方法进行转发，和 DR 模式完全一样，本质上就是执行如下代码
     ```cpp
     NF_HOOK(pf, NF_INET_LOCAL_OUT, (skb), NULL, (rt)->u.dst.dev, dst_output);
     ```
     调用完毕后，就将数据包送往 output 链上，直接执行后续的相关数据包流程，包括邻居子系统填充 mac 信息，最终将数据包发送出去，完成 NAT 请求数据包的转发。
> 注：上述所展示的所有代码逻辑都是在 ip_vs_in 函数内调用完成的，即使后续的 NET_HOOK 相关调用，这时需要注意个小细节，xmit 函数返回的是 NF_STOLEN 值，代表的意思就是说这个数据包我已经处理完毕了，后边的代码就不要再进行处理啦。

**（二）NAT 响应数据包处理流程**

NAT 响应数据包是后端服务器发给 lvs 设备，此时目的 IP 是客户端 IP 地址，需要注册在 `forward` 链上的 `ip_vs_out` 函数来进行处理：

1. NAT 响应数据包处理流程入口函数是 `ip_vs_out`
2. 查找四层协议，确认响应数据包是 lvs 支持的协议，不支持的就会丢弃
3. 根据响应数据包的 5 元组信息（proto，sip，sport，dip，dport），查找连接表，如果查找不到就不再继续 lvs 的流程处理了。
   ```cpp
   /* Check if the packet belongs to an existing entry */
   cp = pp->conn_out_get(af, skb, pp, &iph, iph.len, 0);
   ```
4. 接下来的一系列操作处理函数在 `handle_response`
5. 修改数据包四层信息，调用函数 `tcp_snat_handler`
   ```cpp
   /* 1. 数据包的源端口需要修改为 vport，否则客户端不认识这个连接 */
   tcph->source = cp->vport;
   /* 2. 重新计算 tcp 校验和，计算前需要将 check 字段置为 0 */
   tcph->check = 0;
   tcph->check = csum_tcpudp_magic(cp->caddr.ip, cp->daddr.ip, skb->len - tcphoff, cp->protocol, skb->csum);
   ```
6. 修改数据包三层信息。NAT 需要将响应数据包的源 IP 修改为 vip，这样就能保证这些信息修改对客户端是透明的。
   ```cpp
   /* 1. 修改源 IP 地址为 vip */
   ip_hdr(skb)->saddr = cp->vaddr.ip;
   /* 2. 修改 IP 后，需要重新计算 3 层校验和 */
   ip_send_check(ip_hdr(skb));
   ```
7. 更新 tcp 连接的状态机。注意这时 NAT 状态机表分组选择的是 OUTPUT。
   ```cpp
   /*	OUTPUT */
   /*        sNO, sES, sSS, sSR, sFW, sTW, sCL, sCW, sLA, sLI, sSA	*/
   /*syn*/ {{sSS, sES, sSS, sSR, sSS, sSS, sSS, sSS, sSS, sLI, sSR }},
   /*fin*/ {{sTW, sFW, sSS, sTW, sFW, sTW, sCL, sTW, sLA, sLI, sTW }},
   /*ack*/ {{sES, sES, sSS, sES, sFW, sTW, sCL, sCW, sLA, sES, sES }},
   /*rst*/ {{sCL, sCL, sSS, sCL, sCL, sTW, sCL, sCL, sCL, sCL, sCL }},
   ```
8. **直接返回 NF_ACCEPT**。可以看到这里和之前 NAT 流程是有区别的，这里直接 return 不需要调用 `IP_VS_XMIT`。由于 `ip_vs_out` 函数工作在 `FORWARD` 链上，它已经通过路由查找了，在此函数执行完毕后，会按照内核默认的处理流程去执行，无需 lvs 再做其他的处理和干预了，所以直接返回 NF_ACCEPT 就 ok，接下来会执行:
   ```
   ip_forward_finish -> ip_output -> ip_finish_output() -> ...
   ```
到此就完成了 NAT 响应数据包的处理流程，实现 NAT 模式的转发功能。简单总结如下：
* 查连接表
* 修改源端口和源 IP 地址
* 更新状态机
* 默认流程继续 go ...


## 四、Tunnel 模式数据包处理流程

Tunnel 模式貌似在各公司并不是很常用，我们可以先简单通过下图简单回顾下 Tunnel 的工作原理，然后对比着图示来分析代码实现原理。

![tunnel](https://www.linuxblogs.cn/image/lb007tunnel.png)

我们知道 Tunnel 模式，服务器的响应数据包也是不经过 lvs 的，这点和 DR 有点类似，它只需要处理转发客户端请求的数据包即可。因此 Tunnel 模式只需要在 `INPUT` 连上注册 hook 函数即可，它的入口函数同样是 `ip_vs_in`。

Tunnel 数据包主处理逻辑，在前边的阶段和 DR 模式基本上一样的，所以会叙述的比较简单。只有在 xmit 阶段时，数据包的处理才会有所不同。


* Tunnel 模式下，客户端请求数据包处理的入口函数是 `ip_vs_in`
* 查找四层协议，确认请求数据包是否为 lvs 支持的协议，不支持就会直接丢弃
* 查找相关 svc，调度新的 RS 后端服务器，然后新建连接表项，hash 到连接表
* 更新设置连接相关的状态机，和 DR 模式状态机基本完全一致。
* Tunnel 数据包转发 xmit 阶段处理流程，它对应的 xmit 函数是 `ip_vs_tunnel_xmit`

下边分析 `ip_vs_tunnel_xmit` 函数的相关代码：
* 路由查找。根据 dest 为 daddr 进行路由查找。
* 调整 mtu。因为 Tunnel 模式需要在原有数据包头增加一个 ip 头部，导致数据包体变大，可能会超过原有 mtu 值，因此需要调整 pmtu 相关信息。
  ```cpp
  mtu = dst_mtu(&rt->u.dst) - sizeof(struct iphdr);
  skb_dst(skb)->ops->update_pmtu(skb_dst(skb), mtu);
  ```
* 为 skb 数据结构内存空间调整 headroom，也就是在数据包前边在扩容 IP 头部所占用的空间。
  ```cpp
  skb_realloc_headroom(skb, max_headroom);
  ```
* 填充外部 IP 头部的相关字段信息。
  ```cpp
  iph			=	ip_hdr(skb);
  iph->version	=	4;
  iph->ihl		=	sizeof(struct iphdr)>>2;
  iph->frag_off	=	df;
  iph->protocol	=	IPPROTO_IPIP;
  iph->tos		=	tos;
  /* 1. 目的地址设置的为 dest 服务器地址 */
  iph->daddr	=	rt->rt_dst;
  /* 2. 源地址是根据路由选出的 Local IP 地址 */
  iph->saddr	=	rt->rt_src;
  iph->ttl		=	old_iph->ttl;
  ip_select_ident(iph, &rt->u.dst, NULL);
  ```
* 真正转发数据包阶段。这里调用的不是 `IP_VS_XMIT` 方法，调用的是 `ip_local_out` 函数，通过下边代码就能看出，其实作用和 DR 模式调用 `IP_VS_XMIT` 函数是完全一样的。
  ```cpp
  int __ip_local_out(struct sk_buff *skb)
  {
      struct iphdr *iph = ip_hdr(skb);
      /* 因为 Tunnel 增加了 IP 头部，导致数据包长度发生了变化 */
      iph->tot_len = htons(skb->len);
      ip_send_check(iph);
      return nf_hook(PF_INET, NF_INET_LOCAL_OUT, skb, NULL, skb_dst(skb)->dev, dst_output);
  }
  ```
由此 Tunnel 后续数据包的处理流程和 DR 的情形一样，由内核完成后续的数据包操作。

```
local_out -> ip_output -> ip_finish_output -> ip_finish_output2 -> neigh_hh_output
```
到此为止，Tunnel 模式的数据包转发处理逻辑已经完成。


