---
layout: post
date: "2020-01-07 20:10:12"
title:  "负载均衡 LVS 入门教程详解 - 操作实践"
categories: 负载均衡
tags: lvs
author: 肖邦
---

* content
{:toc}

> 通过之前的内容我们大概对 LVS 有了直观认识，理解了 LVS 的一些基本工作原理。这篇文章通过实践操作的方式，搭建实际测试环境，一步一步配置 LVS 的几种模式。




## 一、实验环境说明


我们知道 LVS 工作模式有 DR、NAT、Tunnel 模式，这篇文章中会针对每个模式进行实践操作，通过实验来更深入理解工作原理。我们需要至少 4 台服务器来进行模拟实验。

* 操作系统
  ```
  CentOS release 6.5 (Final)
  ```
* 内核版本
  ```
  2.6.32-754.15.3.el6.x86_64
  ```
* 相关服务器
  ```bash
  1 台客户端服务器
  1 台负载均衡服务器
  2 台后端真实服务器（模拟负载均衡调度）
  
  注：为了简单，我的几台服务器都是在都一个二层网络，如果有条件的话可以模拟更逼真一些。
  ```

> 注：虚拟机也可以，另外如果资源紧张的话，三台服务器也可（RS一台）


## 二、LVS 相关组件

在进行试验操作之前，有必要先简单

* **IPVS**。LVS 是基于内核态的 netfilter 框架实现的 IPVS 功能，工作在内核态。那么用户是如何配置 VIP 等相关信息并传递到 IPVS 呢，就需要用到了 ipvsadm 工具。
* **ipvsadm 工具**。ipvsadm 是 LVS 用户态的配套工具，可以实现 VIP 和 RS 的增删改查功能。它是基于 `netlink` 或 `raw socket` 方式与内核 LVS 进行通信的，如果 LVS 类比于 netfilter，那么 ipvsadm 就是类似 iptables 工具的地位。
* **keepalived**。ipvsadm 是一个命令行工具，如果 LVS 需要配置的业务非常复杂，ipvadm 就很不方便了。keepalived 最早就是为了 LVS 而生的，非官方开发的，它提供了配置文件的形式配置管理（持久化），服务的增删改查，操作非常方便。另外，keepalived 支持配置虚拟的 VIP，能够实现 LVS 的高可用，实际生产环境中一般离不开它。

虽然 keepalived 使用起来非常方便，不过在实验环境中为了简化步骤，也能够更清楚的理解操作步骤和原理，所有的操作都是用 ipvsadm 命令来进行的。后边也会简单介绍使用 keepalived 进行 LVS 服务的操作管理。


## 三、DR 模式操作实践

**（一）实验环境**

* 客户端：`10.211.102.46`
* 负载均衡：VIP=`10.211.102.47`  DIP=`172.16.100.10`
* 后端1：`172.16.100.20`
* 后端2：`172.16.100.30`

**（二）负载均衡服务器配置**

安装 ipvsadm 工具

```bash
$ yum install ipvsadm
```

配置vip服务

```bash
$ vim lvs_dr.sh
# 内容如下：
vip=10.211.102.47
rs1=172.16.100.20
rs2=172.16.100.30
/sbin/ipvsadm -C                               # 清除原有规则
/sbin/ipvsadm -A -t $vip:8088 -s rr						 # 添加vip:8088的tcp服务
/sbin/ipvsadm -a -t $vip:8088 -r $rs1:8088 -g  # 添加rs1服务器
/sbin/ipvsadm -a -t $vip:8088 -r $rs2:8088 -g  # 添加rs2服务器

# 执行脚本
$ /bin/bash lvs_dr.sh
```

查看服务配置情况

```bash
$ ipvsadm -ln -t 10.211.102.47:8088
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.211.102.47:8088 rr
  -> 172.16.100.20:8088           Route   1      0          0
  -> 172.16.100.30:8088           Route   1      0          0
```

**（三）后端服务器配置**

后端服务器需要安装 nginx 服务

```bash
# 安装 nginx 启动并侦听 8088 端口
$ yum install nginx

# rs 配置 lo 和其它内核参数
$ vim rs_dr.sh  # 内容如下:
vip=10.211.102.47
ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up
echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```

执行脚本：

```bash
$ /bin/bash rs_dr.sh
$ ifconfig lo:0                       # 查看rs的lo:0接口配置信息
lo:0      Link encap:Local Loopback
          inet addr:10.211.102.47  Mask:255.255.255.255
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
```

注：这些步骤需要在两个 RS 上都进行相关操作。

**（四）执行测试**

上述所有步骤执行完毕后，如果一切都顺利的话，执行测试命令会出现如下结果：

```bash
$ curl 10.211.102.47:8088
hello from rs1...
$ curl 10.211.102.47:8088
hello from rs2...
```

实验成功！两次请求分别被调度到了两台的后端服务器上了。感兴趣的话，在后端服务器上抓包观测实际数据包的各个字段是否符合预期。


## 四、DR 配置信息的含义

通过上述步骤，完成最简单的 DR 模式配置。在负载均衡服务器上配置的信息容易理解，而在后端服务器上配置了 lo 和内核参数，下边以问题的方式来解释清楚这些含义。

* 为什么需要在 lo 接口配置 vip ？

> A：我们知道如果服务器收到的数据包 IP 地址不是本机地址（没开启转发模式），就会丢弃。后端服务器收到 DR 模式的数据包，此时目标 IP 是 VIP，服务器会认为此数据包不是发送给本机的，会丢弃。而我们在 lo 接口上配置 VIP 后，服务器就能够正常接收此数据包，发送给相应的应用程序了。因此，在 lo 上配置 vip 能够完成接收数据包并将结果返回给 client。

* 为什么需要配置 arp_ignore 参数？

> A：我们在 lo 上配置了 vip，正常情况下，其它设备发送 vip 的 arp 请求时，除了负载均衡设备会响应 arp 请求之外，后端服务器也会响应 vip 的 arp 请求，这样请求设备不知道哪个是准确的了，反正就是乱了。我们在参数中配置 `arp_ignore=1` 之后，后端服务器只会响应目的 IP 地址为接收网卡上的本地地址的 arp 请求。由于 vip 配置了在 lo 上，所以其它接口收到相关的 arp 请求都会忽略掉，这样保证了 arp 请求正确性。这也说明了为什么 vip 必须配置在 lo 接口上而不是其它接口上了。

* 为什么需要配置 arp_announce 参数？

> 当后端服务器向客户端发送响应数据包时，源地址和目的地址确定，通过目标地址查找路由后，发送网卡也是确认的，也就是源 MAC 地址是确认的，目的 MAC 地址还不确定，需要获取下一跳 IP 对应的 MAC 地址，就需要发送 arp 请求了。发送的 arp 请求目的 IP 就是下一跳的地址，而源 IP 是什么呢？系统通常默认会取数据包的源 IP 作为 arp 的源 IP。我们认真想一下，源 IP 不就是 VIP 么，假设以 VIP 为源 IP 发送 arp 请求，那下一跳（路由器）学习到的 MAC 地址和 IP 地址对应关系就会错乱，因为负载均衡设备上的 VIP 对应的 MAC 地址肯定与这个不同，导致整个系统 arp 紊乱。
>
> 而我们配置参数 `arp_announce=2` 后，操作系统在发送 arp 请求选择源 IP 时，就会忽略数据包的源地址，选择发送网卡上最合适的本地地址作为 arp 请求的源地址。 


## 五、NAT 模式操作实践

**（一）实验环境**

* 客户端：`10.211.102.46`
* 负载均衡：VIP=`10.211.102.47`  DIP=`172.16.100.10`
* 后端1：`172.16.100.20`
* 后端2：`172.16.100.30`

**（二）负载均衡服务器配置**

安装 ipvsadm 并执行相关配置

```bash
$ yum install ipvsadm
$ vim lvs_nat.sh    # 内容如下
echo 1 > /proc/sys/net/ipv4/ip_forward
vip=10.211.102.47
rs1=172.16.100.20
rs2=172.16.100.30
/sbin/ipvsadm -C
/sbin/ipvsadm -A -t $vip:8088 -s rr
/sbin/ipvsadm -a -t $vip:8088 -r $rs1:8088 -m
/sbin/ipvsadm -a -t $vip:8088 -r $rs2:8088 -m

# 执行脚本
$ /bin/bash lvs_nat.sh
```

查看服务配置信息

```bash
$ ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.211.102.47:8088 rr
  -> 172.16.100.20:8088           Masq    1      0          0
  -> 172.16.100.30:8088           Masq    1      0          0
```

**（三）后端服务器配置**

后端服务器安装和配置如下：

```bash
$ yum install nginx    # 安装 nginx，启动并侦听 8088 端口
# vim rs_nat.sh        # 如下内容
route add default gw 172.16.100.10 dev eth0    # 默认网关配置为负载均衡的 DIP

# 执行脚本（两个rs都需要执行）
$ /bin/bash rs_nat.sh
```

**（四）执行测试**

上述所有步骤执行完毕后，如果一切都顺利的话，执行测试命令会出现如下结果：

```bash
$ curl 10.211.102.47:8088
hello from rs1...
$ curl 10.211.102.47:8088
hello from rs2...
```

实验成功！两次请求分别被调度到了两台的后端服务器上了。感兴趣的话，在后端服务器上抓包观测实际数据包的各个字段是否符合预期。

**（五）关于后端服务器设置默认网关**

在后端服务器为何要设置默认网关指向负载均衡设备？由于 NAT 模式是双向流量都经过 LVS 设备，所以后端服务器需要将响应报文发送给 LVS 设备，不过此时的数据包目的 IP 是客户端的 IP，可能是外网的任何一个网段，需要通过设置路由的方式，将数据包转发给 LVS ，因为不可能穷举所有的网段地址，所以必须要设置默认的网关来指向 LVS。最后也要强调一下，负载均衡设备需要开启 ipv4 的 forward 参数。

## 六、Tunnel 模式操作实践

**（一）实验环境**

- 客户端：`10.211.102.46`
- 负载均衡：VIP=`10.211.102.47`  DIP=`172.16.100.10` 
- 后端1：`172.16.100.20` ，注，rs 与 DIP 可以不在一个网段，只要三层通就行。
- 后端2：`172.16.100.30`

**（二）负载均衡服务器配置**

安装 ipvsadm 并执行相关配置

```bash
$ yum install ipvsadm
$ vim lvs_tunnel.sh   # 内容如下
vip=10.211.102.47
rs1=172.16.100.20
rs2=172.16.100.30
/sbin/ipvsadm -C
/sbin/ipvsadm -A -t $vip:8088 -s rr
/sbin/ipvsadm -a -t $vip:8088 -r $rs1:8088 -i
/sbin/ipvsadm -a -t $vip:8088 -r $rs2:8088 -i

## 执行脚本
$ /bin/bash lvs_tunnel.sh
```

查看服务配置信息

```bash
$ ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.211.102.47:8088 rr
  -> 172.16.100.20:8088           Tunnel  1      0          0
  -> 172.16.100.30:8088           Tunnel  1      0          0
```

**（三）后端服务器配置**

后端服务器安装和配置如下：

```bash
$ yum install nginx    # 安装 nginx，启动并侦听 8088 端口
# vim rs_tunnel.sh        # 如下内容
vip=10.211.102.47
modprobe ipip
ifconfig tunl0 $vip broadcast $vip netmask 255.255.255.255 up
echo "0" > /proc/sys/net/ipv4/ip_forward
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
echo "0" > /proc/sys/net/ipv4/conf/all/rp_filter
echo "0" > /proc/sys/net/ipv4/conf/tunl0/rp_filter

# 执行脚本（两个rs都需要执行）
$ /bin/bash rs_tunnel.sh
```

**（四）执行测试**

上述所有步骤执行完毕后，如果一切都顺利的话，执行测试命令会出现如下结果：

```bash
$ curl 10.211.102.47:8088
hello from rs1...
$ curl 10.211.102.47:8088
hello from rs2...
```

实验成功！两次请求分别被调度到了两台的后端服务器上了。感兴趣的话，在后端服务器上抓包观测实际数据包的各个字段是否符合预期。

**（五）为何设置 rp_filter 参数**

为何需要设置 rp_filter 参数？A：默认 rp_filter 参数设置为 1，Linux 会反向校验源 IP 的路由，如果源和目的 IP 路由出口不是一个网卡，就会被丢弃。因为数据包到后端服务器物理网卡后会被转发给虚拟网卡 tunl0 接口，这时进入的网卡是 tunl0，而源 IP 的路由出口网卡肯定不是 tunl0，所以数据包会被丢弃。配置 rp_filter 参数为 0，就相当于关闭了这个校验。


## 七、使用keepalived配置LVS

上述的所有操作都是通过 ipvsadm 命令来实现的，虽然也能完成相应的功能，但也存在如下的问题：

* 命令行操作和变更，很不方便。
* 配置信息不方便持久化。
* 假设一个 RealServer 宕机或无响应，LVS 仍会将请求转发至后端，导致业务不可用。
* 存在单点问题，如果 LVS 服务器宕机，则全部服务不可用。

keepalived 本身就是为 LVS 而开发的，能够优雅的解决上述问题。

**（一）实验环境**

- 客户端：`10.211.102.46`
- 负载均衡1：VIP=`10.211.102.47`  DIP=`172.16.100.10` 
- 负载均衡2：VIP=`10.211.102.48`  DIP=`172.16.100.40` 
- 后端1：`172.16.100.20` ，注，rs 与 DIP 可以不在一个网段，只要三层通就行。
- 后端2：`172.16.100.30`

**（二）安装配置keepalvied**

负载均衡设备需要安装 keepalived 软件
```bash
$ yum install keepalived
```
keepalived master 配置文件
```bash
$ vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.211.102.47
    }
}

virtual_server 10.211.102.47 8088 {
    delay_loop 3
    lb_algo wrr
    lb_kind DR
    protocol TCP

    real_server 172.16.100.20 8088 {
        weight 1
        TCP_CHECK {
            connect_timeout 10
            connect_port 8088
        }
    }

    real_server 172.16.100.30 8088 {
        weight 1
        TCP_CHECK {
            connect_timeout 10
            connect_port 8088
        }
    }
}
```
backup 结点配置文件
```bash
# 拷贝主节点的配置文件keepalived.conf，然后修改如下内容：
state MASTER -> state BACKUP
priority 100 -> priority 90
```
启动 keepalvied 软件
```bash
$ service keepalived start
```

**（三）执行测试**

* **健康检查**。手动关闭 rs2，执行测试命令，发现 lvs 会一直将请求转发至 rs1。恢复 rs2，lvs 很快就会感知并将 rs2 加入到集群列表。
* **高可用**。手动 stop 掉 lvs1，会发现 vip 漂移到 lvs2 上，执行测试命令，正常工作。


## 八、LVS 的几种调度算法

在前边的一些实验过程中，会发现 ipvsadm 命令最后边的 `-s rr` ，就表示的是调度算法，其中 rr 就是最简单的轮询调度算法，下边先简单介绍负载均衡 LVS 支持的几种调度算法。

* `轮询rr` - 这种算法是最简单的，就是按依次循环的方式将请求调度到不同的服务器上，该算法最大的特点就是简单。轮询算法假设所有的服务器处理请求的能力都是一样的，调度器会将所有的请求平均分配给每个真实服务器，不管后端 RS 配置和处理能力，非常均衡地分发下去。
* `加权轮询wrr` - 这种算法比 rr 的算法多了一个权重的概念，可以给 RS 设置权重，权重越高，那么分发的请求数越多。实际生产环境的服务器配置可能不同，根据服务器配置适当设置权重，可以有效利用服务器资源。
* `最少连接lc` - 这个算法会根据后端 RS 的连接数来决定把请求分发给谁，比如 RS1 连接数比 RS2 连接数少，那么请求就优先发给 RS1 
* `加权最少链接 wlc` - 这个算法比 lc 多了一个权重的概念。
* `基于局部性的最少连接调度算法 lblc` - 这个算法是请求数据包的目标 IP 地址的一种调度算法，该算法先根据请求的目标 IP 地址寻找最近的该目标 IP 地址所有使用的服务器，如果这台服务器依然可用，并且有能力处理该请求，调度器会尽量选择相同的服务器，否则会继续选择其它可行的服务器。
* `复杂的基于局部性最少的连接算法 lblcr` - 记录的不是要给目标 IP 与一台服务器之间的连接记录，它会维护一个目标 IP 到一组服务器之间的映射关系，防止单点服务器负载过高。
* `目标地址散列调度算法 dh` - 该算法是根据目标 IP 地址通过散列函数将目标 IP 与服务器建立映射关系，出现服务器不可用或负载过高的情况下，发往该目标 IP 的请求会固定发给该服务器。
* `源地址散列调度算法 sh` - 与目标地址散列调度算法类似，但它是根据源地址散列算法进行静态分配固定的服务器资源。

