---
layout: post
date: "2020-01-13 18:55:11"
title:  "负载均衡 LVS 入门教程详解 - 调度算法"
categories: 负载均衡
tags: lvs
author: 肖邦
---

* content
{:toc}
> 在前边简单总结了连接表相关的内容，连接表贯穿了 lvs 核心主流程，而这篇内容来介绍 lvs 的另一核心内容，就是调度算法。




## （一）调度算法介绍
负载均衡器 lvs 本质上就是负责将客户端的数据包流程正确、均衡的转发给后端的真实服务器。
* **正确**。意思是要保证属于同一个连接会话的数据包转发给固定的一个后端服务器，保证连接的正常交互。
* **均衡**。均衡并不是说平均，而是通过某种调度方式根据服务器的硬件条件或运行数据，将数据包合理的、均衡的分给后端的真实服务器。简单的说，高配置的服务器可以多分配些流量；那些承载流量较少的服务器，我们可以适当多分点流量过去，根据实际情况让后端服务器集群达到服务器能力上的均衡，要保证不会出现某台服务器已经严重 **超载** 而其他服务器却非常 **空闲** 这种现象。

不同场景下可能需要不一样的调度算法，目前 lvs 支持 10 种调度算法用于满足不同场景需求，如下简述：

* `rr` - 轮询调度算法
* `wrr` - 加权轮询调度
* `sh` - 源地址哈希调度
* `dh` - 目的地址哈希调度
* `lc` - 最少连接调度
* `wlc` - 加权最少连接
* `sed` - 最少期望延迟调度算法
* `nq` - 永不排队调度算法
* `lblc` - 基于局部性的最少连接
* `lblcr` - 带复制的基于局部性最少链接

在 Linux 系统中可以通过下列命令查看支持哪些模式
```bash
$ grep -i 'IPVS SCHEDULER' -A 12 /boot/config-`uname -r`
```

下边的内容会从 lvs 代码的角度来总结大部分常用的调度算法实现原理和应用场景。


## （二）调度模块的基础

lvs 对于每种调度算法都是以一个 .c 文件来组织，编译后以内核模块 ko 的形式存在，在 lvs 中会被使用到的调度算法模块 *.ko 都会被自动加载进内核。

在 lvs 初始化阶段，只会加载 ip_vs.ko 主流程代码，默认是不加载各个调度算法对应的 ko 模块的。只有在管理员添加服务 svc 配置时，该配置中设置了的调度算法模块才被会加载进内核，此时会调用该模块的初始化阶段代码，进行模块的注册以及与 svc 的绑定。

先看下调度算法对应的数据结构
```cpp
struct ip_vs_scheduler {
	struct list_head	n_list;		/* 组织链表结构的关键字段 */
	char			*name;		    /* 调度算法名称 */
	atomic_t		refcnt;		    /* 引用计数器 */
	struct module		*module;

    /**
     * 调度算法对应的操作函数
     */

    /* 模块初始化绑定时会被调用 */
	int (*init_service)(struct ip_vs_service *svc); 
	/* 在 unbind 时会被调用，一般是与初始化相反的内容 */
	int (*done_service)(struct ip_vs_service *svc); 
	/* 在配置发生变化时，需要更新调度器相关的数据信息，此时会被调用 */
	int (*update_service)(struct ip_vs_service *svc);

	/* 调度器的核心代码：从已知的服务器列表中选择一个正确合适的服务器 */
	struct ip_vs_dest* (*schedule)(struct ip_vs_service *svc, const struct sk_buff *skb);
};
```

在内核模块加载时，会调用模块的初始化函数 `ip_vs_xx_init()`，一般在 init 函数中都会进行注册相关的操作，调用如下函数进行注册：

```cpp
static struct ip_vs_scheduler ip_vs_xx_scheduler =
{
	.name           =   "xx",
	.refcnt         =   ATOMIC_INIT(0),
	.module         =   THIS_MODULE,
	.n_list         =   LIST_HEAD_INIT(ip_vs_xx_scheduler.n_list),
	.init_service   =	ip_vs_xx_init_svc,
	.done_service   =	ip_vs_xx_done_svc,
	.update_service =   ip_vs_xx_update_svc,
	.schedule       =   ip_vs_xx_schedule,
};
register_ip_vs_scheduler(&ip_vs_xx_scheduler);
/**
 * 具体是如何注册就不再详细说明，可以简单理解就是，把各个调度算法平均挂载到了
 * 一个哈希表中了。
 */
```

注册完毕后的 scheduler 就在 lvs 内存信息中保存，后续有管理员添加该调度算法的 svc 时，就会调用 `ip_vs_bind_scheduler` 函数，将 svc 的调度器设置为该名称对应的调度算法，也就是 svc 能够持有该调度算法的对应所有相关信息，需要使用调度器时，直接调用 .schedule 回调函数即可。


## （三）经典的轮询调度算法（rr）

**轮询调度**（Round Robin）算法就是以轮询的方式依次将新建连接的请求调度到不同的服务器上，完全不管服务器的负载、性能配置等信息。该算法优点是简单，无需记录后端服务器的连接数，它是一种无状态的调度算法。

```cpp
static struct ip_vs_dest *
ip_vs_rr_schedule(struct ip_vs_service *svc, const struct sk_buff *skb)
{
	write_lock(&svc->sched_lock);
	p = (struct list_head *)svc->sched_data;
	p = p->next;
	q = p;
	do {
		if (q == &svc->destinations) {
			q = q->next;
			continue;
		}

		dest = list_entry(q, struct ip_vs_dest, n_list);
		if (!(dest->flags & IP_VS_DEST_F_OVERLOAD) &&
		    atomic_read(&dest->weight) > 0)
			goto out;
		q = q->next;
	} while (q != p);
	write_unlock(&svc->sched_lock);
	return NULL;

  out:
	svc->sched_data = q;
	write_unlock(&svc->sched_lock);
	return dest;
}
```

代码比较简单，就是单纯从 svc->dests 列表中轮询取下一个 RS 服务器，当然要排除掉那些过载和权重为 0 的服务器，将轮询到的服务器 dest 返回即可。这种调度算法简单，适合那种需求简单，而且后端服务器配置性能基本一致的情况下比较合适。


## （四）基于权重的轮询调度算法（wrr）

加权轮询调度算法是对 rr 算法上的优化，是基于权重的轮询调度算法。试想有一组服务器，每个服务器都是不同时期采购的，性能配置相差较大，如果采用简单的轮询（rr）算法，那么性能最差的服务器很快就过载了，而性能好的服务器还很空闲，对于这样一组服务器采用 rr 算法的话，就不能充分利用高配置的服务器性能了。我们可以通过 wrr 调度算法，为每台服务器设置对应的权重比例因数，性能高的权重就设置得大一些，性能差的就设置得小一些，这样就能充分灵活的利用这组性能差异较大的服务器了。

```cpp
static struct ip_vs_dest *
ip_vs_wrr_schedule(struct ip_vs_service *svc, const struct sk_buff *skb)
{
	struct ip_vs_dest *dest;
	struct ip_vs_wrr_mark *mark = svc->sched_data;
	struct list_head *p;

	write_lock(&svc->sched_lock);
	p = mark->cl;
	while (1) {
		if (mark->cl == &svc->destinations) {
			if (mark->cl == mark->cl->next) {
				dest = NULL;
				goto out;
			}
			mark->cl = svc->destinations.next;
			mark->cw -= mark->di;
			if (mark->cw <= 0) {
				mark->cw = mark->mw;
				if (mark->cw == 0) {
					mark->cl = &svc->destinations;
					dest = NULL;
					goto out;
				}
			}
		} else
			mark->cl = mark->cl->next;

		if (mark->cl != &svc->destinations) {
			dest = list_entry(mark->cl, struct ip_vs_dest, n_list);
			if (!(dest->flags & IP_VS_DEST_F_OVERLOAD) &&
			    atomic_read(&dest->weight) >= mark->cw) {
				break;
			}
		}

		if (mark->cl == p && mark->cw == mark->di) {
			dest = NULL;
			goto out;
		}
	}
  out:
	write_unlock(&svc->sched_lock);
	return dest;
}
```

上述代码比 rr 稍复杂些，基本原理实现不很复杂，不过只用文字叙述还是有些苍白无力，还是举例来说明吧。
1. 假设这里有 3 台后端服务器，权重分别是 A=100 B=200 C=400
2. 求 3 个服务器权重的最小公约数，为 100
3. 设置 M=400，每轮过后减去最小公约数 100
4. 这里是 3 台服务器，所以每一轮分别从三台权重依次与 M 值进行比较，如果大于等于 M 就选中此服务器，每一轮后 M 值减 100

通过此方法步骤得出如下大概的图示：
```
   A  B  C
1. x  x  v  // 第一轮，三台中只有 C 权重大于等于 M=400
2. x  x  v  // 第二轮，三台中只有 C 权重大于等于 M=300
3. x  v  v  // 第三轮，三台中有 B C 权重大于等于 M=200
4. v  v  v  // 第四轮，三台中 A B C 权重大于等于 M=100
```
通过上述举例，可以明显看出总调度了 7 台服务器，而 C 占了 4 次，而 B 占了 2 次，A 占了 1 次，整体上就实现了三台服务器的调度比例为 `1:2:4` 效果了。

加权轮询调度算法解决了 rr 算法针对性能配置不一的服务器不能充分利用的问题，这种方式在生产环境中使用的比较多。

## （五）源地址哈希调度（sh）

源地址散列调度它根据请求的源 IP 地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。

```cpp
static struct ip_vs_dest *
ip_vs_sh_schedule(struct ip_vs_service *svc, const struct sk_buff *skb)
{
	struct ip_vs_dest *dest;
	struct ip_vs_sh_bucket *tbl;
	struct ip_vs_iphdr iph;

	ip_vs_fill_iphdr(svc->af, skb_network_header(skb), &iph);

	tbl = (struct ip_vs_sh_bucket *)svc->sched_data;
	dest = ip_vs_sh_get(svc->af, tbl, &iph.saddr);
	if (!dest
	    || !(dest->flags & IP_VS_DEST_F_AVAILABLE)
	    || atomic_read(&dest->weight) <= 0
	    || is_overloaded(dest)) {
		IP_VS_ERR_RL("SH: no destination available\n");
		return NULL;
	}
	return dest;
}
```

分析上述代码，根据数据包的 src-ip 计算哈希值，根据哈希值映射关系找到一个后端的服务器。这个算法有个特点，就是某个客户端的所有连接都会固定分配调度到某个后端服务器上，对于某些特殊业务需要长时间保持会话信息的需求会起到不错的效果。


## （六）目标地址哈希调度（dh）

目标地址哈希调度算法是针对目标 IP 地址的负载均衡，但它是一种静态映射算法，通过一个散列（Hash）函数将一个目标 IP 地址映射到一台服务器。

```cpp
static struct ip_vs_dest *
ip_vs_dh_schedule(struct ip_vs_service *svc, const struct sk_buff *skb)
{
	struct ip_vs_dest *dest;
	struct ip_vs_dh_bucket *tbl;
	struct ip_vs_iphdr iph;
	ip_vs_fill_iphdr(svc->af, skb_network_header(skb), &iph);
	tbl = (struct ip_vs_dh_bucket *)svc->sched_data;
	dest = ip_vs_dh_get(svc->af, tbl, &iph.daddr);
	if (!dest
	    || !(dest->flags & IP_VS_DEST_F_AVAILABLE)
	    || atomic_read(&dest->weight) <= 0
	    || is_overloaded(dest)) {
		return NULL;
	}
	return dest;
}
```

代码和 sh 基本是一样的，只不过哈希计算是通过目的 IP。刚开始对这种调度算法存在一些疑惑，通过目的 IP 哈希的话，那所有请求访问的目的 IP 不都是 vip 么，那就算出来的哈希值不都一样吗？也就是说要转发给后端固定一台服务器，这肯定是理解的有问题的。其实除了 svc 方式来标识服务外，还可以通过 fwmark 的方式来标识。也在网上查一些资料，据说是可以在后端缓存（Redis）的应用场景下使用。通过目的 IP 哈希计算可以将访问相同资源的请求都缓存在同一台缓存服务器上，防止同一资源缓存在不同服务器上，这样有效利用率就会很低。


## （七）最小连接调度算法（lc）

最小连接调度算法（lc）是将新的连接请求分配调度到当前连接数最小的服务器上。这是一种动态调度算法，它通过当前服务器活跃连接数估计服务器的负载情况，将请求连接分配到负载最小的服务器上。

```cpp
static struct ip_vs_dest *
ip_vs_lc_schedule(struct ip_vs_service *svc, const struct sk_buff *skb)
{
	struct ip_vs_dest *dest, *least = NULL;
	unsigned int loh = 0, doh;
	list_for_each_entry(dest, &svc->destinations, n_list) {
		if ((dest->flags & IP_VS_DEST_F_OVERLOAD) ||
		    atomic_read(&dest->weight) == 0)
			continue;
		doh = ip_vs_lc_dest_overhead(dest);
		if (!least || doh < loh) {
			least = dest;
			loh = doh;
		}
	}
	// ....省略调度失败的日志打印代码....
	return least;
}

```
分析上述代码，逻辑很简单，调度代码通过对比当前所有 rs 服务器的当前活跃连接数，选择一个最小的连接数的服务器作为本次调度目标。那这时你会不会有这样的疑问，每次平均的将新建连接的分配给各个服务器，为何会存在各服务器连接数较大差异呢？其实仔细思考一下就能想通，每个连接承载的数据内容都不同，有的可能是长连接，有的可能访问的资源是个大文件因此连接需要的时间少长一些，种种不同的情况都可能早就各个服务器上的连接数产生较大差异，用 lc 调度算法就能解决这个问题。


## （八）加权最小连接调度算法（wlc）

加权最小连接调度算法是最小连接调度的优化或功能增强超集，各服务器用相应的权重表示其处理性能。加权最小连接调度在调度新连接时尽可能使服务器的已建立连接数和其权值成比例。

```cpp
static struct ip_vs_dest *
ip_vs_wlc_schedule(struct ip_vs_service *svc, const struct sk_buff *skb)
{
	struct ip_vs_dest *dest, *least;
	unsigned int loh, doh;
	list_for_each_entry(dest, &svc->destinations, n_list) {
		if (!(dest->flags & IP_VS_DEST_F_OVERLOAD) &&
		    atomic_read(&dest->weight) > 0) {
			least = dest;
			loh = ip_vs_wlc_dest_overhead(least);
			goto nextstage;
		}
	}
	IP_VS_ERR_RL("WLC: no destination available\n");
	return NULL;

  nextstage:
	list_for_each_entry_continue(dest, &svc->destinations, n_list) {
		if (dest->flags & IP_VS_DEST_F_OVERLOAD)
			continue;
		doh = ip_vs_wlc_dest_overhead(dest);
		if (loh * atomic_read(&dest->weight) >
		    doh * atomic_read(&least->weight)) {
			least = dest;
			loh = doh;
		}
	}
	return least;
}
```
上述代码逻辑很 lc 算法很像，只是在对比计算时，将连接数和权重的乘积作为对比数据，这样就能将权重标识服务器的性能，根据性能的指标将新建连接分配给所谓的最小连接的服务器上。


## （九）其他类型的调度算法

* **基于局部性的最少链接（lblc）**。基于局部性的最少连接算法是针对请求报文的目标 IP 地址的负载均衡调度，主要用于 Cache 集群系统，因为 Cache 集群中客户请求报文的目标 IP 地址是变化的，这里假设任何后端服务器都可以处理任何请求，算法的设计目标在服务器的负载基本平衡的情况下，将相同的目标 IP 地址的请求调度到同一个台服务器，来提高服务器的访问局部性和主存 Cache 命中率，从而调整整个集群系统的处理能力。
* **基于局部性的带复制功能的最少连接（lblcr）**。基于局部性的带复制功能的最少连接调度算法也是针对目标 IP 地址的负载均衡，该算法根据请求的目标 IP 地址找出该目标 IP 地址对应的服务器组，按 “最小连接” 原则从服务器组中选出一台服务器，若服务器没有超载，将请求发送到该服务器；若服务器超载，则按“最小连接”原则从这个集群中选出一台服务器，将该服务器加入到服务器组中，将请求发送到该服务器。同时，当该服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除， 以降低复制的程度。
* **最少期望延迟（sed）**。不考虑非活动连接，谁的权重大，我们优先选择权重大的服务器来接收请求，但会出现问题，就是权重比较大的服务器会很忙，但权重相对较小的服务器很闲，甚至会接收不到请求，所以便有了下面的算法 nq。
* **永不排队（nq）**。在上面我们说明了，由于某台服务器的权重较小，比较空闲，甚至接收不到请求，而权重大的服务器会很忙，所此算法是sed改进，就是说不管你的权重多大都会被分配到请求。简单说，无需队列，如果有台 real server 的连接数为 0 就直接分配过去，不需要在进行 sed 运算。

这些调度算法在平时用的较少，就不再再从代码层面详细分析了。


