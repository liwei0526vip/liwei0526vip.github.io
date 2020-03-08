---
layout: post
date: "2020-01-04 19:55:12"
title:  "负载均衡 LVS 入门教程详解 - 基础原理"
categories: 负载均衡
tags: lvs
author: 肖邦
---

* content
{:toc}


> LVS 是由章文嵩博士开源的负载均衡系统，它现在是标准内核的一部分，它具备可靠性、高性能、可扩展性和可操作性，从而以低廉的成本实现最优的性能。




## 一、netfilter基本原理

LVS 是基于 Linux 内核中 netfilter 框架实现的负载均衡系统，所以要学习 LVS 之前必须要先简单了解 netfilter 基本工作原理。netfilter 其实很复杂也很重要，平时我们说的 Linux 防火墙就是 netfilter，不过我们平时操作的都是 iptables，iptables 只是用户空间编写和传递规则的工具而已，真正工作的是 netfilter。通过下图可以简单了解下 netfilter 的工作机制：

![netfilter 基本原理图](https://raw.githubusercontent.com/liwei0526vip/blogimg/master/lb003netfilter.png)

netfilter 是内核态的 Linux 防火墙机制，作为一个通用、抽象的框架，提供了一整套的 hook 函数管理机制，提供诸如数据包过滤、网络地址转换、基于协议类型的连接跟踪的功能。

通俗点讲，就是 netfilter 提供一种机制，可以在数据包流经过程中，根据规则设置若干个关卡（hook 函数）来执行相关的操作。netfilter 总共设置了 5 个点，包括：PREROUTING、INPUT、FORWARD、OUTPUT、POSTROUTING

* `PREROUTING` ：刚刚进入网络层，还未进行路由查找的包，通过此处
* `INPUT` ：通过路由查找，确定发往本机的包，通过此处
* `FORWARD` ：经路由查找后，要转发的包，在POST_ROUTING之前
* `OUTPUT` ：从本机进程刚发出的包，通过此处
* `POSTROUTING` ：进入网络层已经经过路由查找，确定转发，将要离开本设备的包，通过此处

当一个数据包进入网卡，经过链路层之后进入网络层就会到达 PREROUTING，接着根据目标 IP 地址进行路由查找，如果目标 IP 是本机，数据包继续传递到 INPUT 上，经过协议栈后根据端口将数据送到相应的应用程序；应用程序处理请求后将响应数据包发送到 OUTPUT 上，最终通过 POSTROUTING 后发送出网卡。如果目标 IP 不是本机，而且服务器开启了 forward 参数，就会将数据包递送给 FORWARD 上，最后通过 POSTROUTING 后发送出网卡。

## 二、LVS基本原理

LVS 是基于 netfilter 框架，主要工作于 INPUT 链上，在 INPUT 上注册 `ip_vs_in` HOOK 函数，进行 IPVS 主流程，大概原理如图所示：

![LVS 基本原理图](https://raw.githubusercontent.com/liwei0526vip/blogimg/master/lb004ipvs.png)


* 当用户访问 www.sina.com.cn 时，用户数据通过层层网络，最后通过交换机进入 LVS 服务器网卡，并进入内核网络层。
* 进入 PREROUTING 后经过路由查找，确定访问的目的 VIP 是本机 IP 地址，所以数据包进入到 INPUT 链上
* IPVS 是工作在 INPUT 链上，会根据访问的 `vip+port` 判断请求是否 IPVS 服务，如果是则调用注册的 IPVS HOOK 函数，进行 IPVS 相关主流程，强行修改数据包的相关数据，并将数据包发往 POSTROUTING 链上。
* POSTROUTING 上收到数据包后，根据目标 IP 地址（后端服务器），通过路由选路，将数据包最终发往后端的服务器上。

开源 LVS 版本有 3 种工作模式，每种模式工作原理截然不同，说各种模式都有自己的优缺点，分别适合不同的应用场景，不过最终本质的功能都是能实现均衡的流量调度和良好的扩展性。主要包括以下三种模式

* DR 模式
* NAT 模式
* Tunnel 模式

另外必须要说的模式是 FullNAT，这个模式在开源版本中是模式没有的，[代码](http://kb.linuxvirtualserver.org/wiki/IPVS_FULLNAT_and_SYNPROXY) 没有合并进入内核主线版本，后面会有专门章节详细介绍 FullNAT 模式。下边分别就 DR、NAT、Tunnel 模式分别详细介绍原理。

## 三、DR 模式实现原理

LVS 基本原理图（上图）中描述的比较简单，表述的是比较通用流程。下边会针对 DR 模式的具体实现原理，详细的阐述 DR 模式是如何工作的。

![LVS DR 模式原理图](https://raw.githubusercontent.com/liwei0526vip/blogimg/master/lb005dr.png)

其实 DR 是最常用的工作模式，因为它的强大的性能。下边以一次请求和响应数据流的过程来描述 DR 模式的具体原理

**（一）实现原理过程**


* ① 当客户端请求 www.sina.com.cn 主页，经过 DNS 解析到 IP 后，向新浪服务器发送请求数据，数据包经过层层网络到达新浪负载均衡 LVS 服务器，到达 LVS 网卡时的数据包：源 IP 是客户端 IP 地址 CIP，目的 IP 是新浪对外的服务器 IP 地址，也就是 VIP；此时源 MAC 地址是 CMAC，其实是 LVS 连接的路由器的 MAC 地址（为了容易理解记为 CMAC），目标 MAC 地址是 VIP 对应的 MAC，记为 VMAC。

* ② 数据包到达网卡后，经过链路层到达 PREROUTING 位置（刚进入网络层），查找路由发现目的 IP 是 LVS 的 VIP，就会递送到 INPUT 链上，此时数据包 MAC、IP、Port 都没有修改。

* ③ 数据包到达 INPUT 链，INPUT 是 LVS 主要工作的位置。此时 LVS 会根据目的 IP 和 Port 来确认是否是 LVS 定义的服务，如果是定义过的 VIP 服务，就会根据配置的 Service 信息，从 RealServer 中选择一个作为后端服务器 RS1，然后以 RS1 作为目标查找 Out 方向的路由，确定一下跳信息以及数据包要通过哪个网卡发出。最后将数据包通过 INET_HOOK 到 OUTPUT 链上（Out 方向刚从四层进入网络层）。

* ④ 数据包通过 POSTROUTING 链后，从网络层转到链路层，将目的 MAC 地址修改为 RealServer 服务器 MAC 地址，记为 RMAC；而源 MAC 地址修改为 LVS 与 RS 同网段的 selfIP 对应的 MAC 地址，记为 DMAC。此时，数据包通过交换机转发给了 RealServer 服务器（注：为了简单图中没有画交换机）。

* ⑤ 请求数据包到达 RealServer 服务器后，链路层检查目的 MAC 是自己网卡地址。到了网络层，查找路由，目的 IP 是 VIP（lo 上配置了 VIP），判定是本地主机的数据包，经过协议栈后拷贝至应用程序（比如这里是 nginx 服务器），nginx 响应请求后，产生响应数据包。以目的 VIP 为 dst 查找 Out 路由，确定吓一跳信息和发送网卡设备信息，发送数据包。此时数据包源、目的 IP 分别是 VIP、CIP，而源 MAC 地址是 RS1 的 RMAC，目的 MAC 是下一跳（路由器）的 MAC 地址，记为 CMAC（为了容易理解，记为 CMAC）。然后数据包通过 RS 相连的路由器转发给真正客户端。

> 从整个过程可以看出，DR 模式 LVS 逻辑非常简单，数据包通过路由方式直接转发给 RS，而且响应数据包是由 RS 服务器直接发送给客户端，不经过 LVS。我们知道一般请求数据包会比较小，响应报文较大，经过 LVS 的数据包基本上都是小包，上述几条因素是 LVS 的 DR 模式性能强大的主要原因。


**（二）优缺点和使用场景**

* DR 模式的优点
> a. 响应数据不经过 lvs，性能高
> b. 对数据包修改小，信息保存完整（携带客户端源 IP）

* DR 模式的缺点
> a. lvs 与 rs 必须在同一个物理网络（不支持跨机房）
> b. rs 上必须配置 lo 和其它内核参数
> c. 不支持端口映射

* DR 模式的使用场景
> 如果对性能要求非常高，可以首选 DR 模式，而且可以透传客户端源 IP 地址。


## 四、NAT 模式实现原理

lvs 的第 2 种工作模式是 NAT 模式，下图详细介绍了数据包从客户端进入 lvs 后转发到 rs，后经 rs 再次将响应数据转发给 lvs，由 lvs 将数据包回复给客户端的整个过程。

![LVS DR 模式原理图](https://raw.githubusercontent.com/liwei0526vip/blogimg/master/lb006nat.png)

**（一）实现原理与过程**


* ① 用户请求数据包经过层层网络，到达 lvs 网卡，此时数据包源 IP 是 CIP，目的 IP 是 VIP。
* ② 经过网卡进入网络层 prerouting 位置，根据目的 IP 查找路由，确认是本机 IP，将数据包转发到 INPUT 上，此时源、目的 IP 都未发生变化。
* ③ 到达 lvs 后，通过目的 IP 和目的 port 查找是否为 IPVS 服务。若是 IPVS 服务，则会选择一个 RS 作为后端服务器，将数据包目的 IP 修改为 RIP，并以 RIP 为目的 IP 查找路由信息，确定下一跳和出口信息，将数据包转发至 output 上。
* ④ 修改后的数据包经过 postrouting 和链路层处理后，到达 RS 服务器，此时的数据包源 IP 是 CIP，目的 IP 是 RIP。
* ⑤ 到达 RS 服务器的数据包经过链路层和网络层检查后，被送往用户空间 nginx 程序。nginx 程序处理完毕，发送响应数据包，由于 RS 上默认网关配置为 lvs 设备 IP，所以 nginx 服务器会将数据包转发至下一跳，也就是 lvs 服务器。此时数据包源 IP 是 RIP，目的 IP 是 CIP。
* ⑥ lvs 服务器收到 RS 响应数据包后，根据路由查找，发现目的 IP 不是本机 IP，且 lvs 服务器开启了转发模式，所以将数据包转发给 forward 链，此时数据包未作修改。
* ⑦ lvs 收到响应数据包后，根据目的 IP 和目的 port 查找服务和连接表，将源 IP 改为 VIP，通过路由查找，确定下一跳和出口信息，将数据包发送至网关，经过复杂的网络到达用户客户端，最终完成了一次请求和响应的交互。

> NAT 模式双向流量都经过 LVS，因此 NAT 模式性能会存在一定的瓶颈。不过与其它模式区别的是，NAT 支持端口映射，且支持 windows 操作系统。

**（二）优点、缺点与使用场景**

* NAT 模式优点
> a. 能够支持 windows 操作系统
> b. 支持端口映射。如果 rs 端口与 vport 不一致，lvs 除了修改目的 IP，也会修改 dport 以支持端口映射。

* NAT 模式缺点
> a. 后端 RS 需要配置网关
> b. 双向流量对 lvs 负载压力比较大

* NAT 模式的使用场景
> 如果你是 windows 系统，使用 lvs 的话，则必须选择 NAT 模式了。


## 五、Tunnel 模式实现原理

Tunnel 模式在国内使用的比较少，不过据说腾讯使用了大量的 Tunnel 模式。它也是一种单臂的模式，只有请求数据会经过 lvs，响应数据直接从后端服务器发送给客户端，性能也很强大，同时支持跨机房。下边继续看图分析原理。

![Tunnel 原理图](https://raw.githubusercontent.com/liwei0526vip/blogimg/master/lb007tunnel.png)

**（一）实现原理与过程**


* ① 用户请求数据包经过多层网络，到达 lvs 网卡，此时数据包源 IP 是 cip，目的 ip 是 vip。
* ② 经过网卡进入网络层 prerouting 位置，根据目的 ip 查找路由，确认是本机 ip，将数据包转发到 input 链上，到达 lvs，此时源、目的 ip 都未发生变化。
* ③ 到达 lvs 后，通过目的 ip 和目的 port 查找是否为 IPVS 服务。若是 IPVS 服务，则会选择一个 rs 作为后端服务器，以 rip 为目的 ip 查找路由信息，确定下一跳、dev 等信息，然后 IP 头部前边额外增加了一个 IP 头（以 dip 为源，rip 为目的 ip），将数据包转发至 output 上。
* ④ 数据包根据路由信息经最终经过 lvs 网卡，发送至路由器网关，通过网络到达后端服务器。
* ⑤ 后端服务器收到数据包后，ipip 模块将 Tunnel 头部卸载，正常看到的源 ip 是 cip，目的 ip 是 vip，由于在 tunl0 上配置 vip，路由查找后判定为本机 ip，送往应用程序。应用程序 nginx 正常响应数据后以 vip 为源 ip，cip 为目的 ip 数据包发送出网卡，最终到达客户端。

> Tunnel 模式具备 DR 模式的高性能，又支持跨机房访问，听起来比较完美了。不过国内运营商有一定特色性，比如 RS 的响应数据包的源 IP 为 VIP，VIP 与后端服务器有可能存在跨运营商的情况，有可能被运营商的策略封掉。Tunnel 在生产环境确实没有使用过，在国内推行 Tunnel 可能会有一定的难度吧。

**（二）优点、缺点与使用场景**

* Tunnel 模式的优点
> a. 单臂模式，对 lvs 负载压力小
> b. 对数据包修改较小，信息保存完整
> c. 可跨机房（不过在国内实现有难度）

* Tunnel 模式的缺点
> a. 需要在后端服务器安装配置 ipip 模块
> b. 需要在后端服务器 tunl0 配置 vip
> c. 隧道头部的加入可能导致分片，影响服务器性能
> d. 隧道头部 IP 地址固定，后端服务器网卡 hash 可能不均
> e. 不支持端口映射

* Tunnel 模式的使用场景
> 理论上，如果对转发性能要求较高，且有跨机房需求，Tunnel 可能是较好的选择。


## 六、涉及的概念术语

上述内容中涉及到很多术语或缩写，这里简单解释下具体的含义，便于理解。

> * `CIP`：Client IP，表示的是客户端 IP 地址。
> * `VIP`：Virtual IP，表示负载均衡对外提供访问的 IP 地址，一般负载均衡 IP 都会通过 Virtual IP 实现高可用。
> * `RIP`：RealServer IP，表示负载均衡后端的真实服务器 IP 地址。
> * `DIP`：Director IP，表示负载均衡与后端服务器通信的 IP 地址。
> * `CMAC`：客户端的 MAC 地址，准确的应该是 LVS 连接的路由器的 MAC 地址。
> * `VMAC`：负载均衡 LVS 的 VIP 对应的 MAC 地址。
> * `DMAC`：负载均衡 LVS 的 DIP 对应的 MAC 地址。
> * `RMAC`：后端真实服务器的 RIP 地址对应的 MAC 地址。


基础性的原理知识，暂时分析理解到这里就可以了，后边内容会尽量以实践的方式来进一步了解 LVS 的工作原理和基本操作使用方式。



