---
layout: post
title:  "ipvsadm 使用手册"
categories: 负载均衡
tags: ipvsadm lvs
author: 肖邦
---

* content
{:toc}

> ipvsadm 是用来管理 LVS 集群配置的，运行在用户态空间，而 LVS 是运行在内核态的。ipvsadm 通过调用 `libipvs` 库，以 `raw socket` 或 `netlink` 的通信方式向 LVS 下发配置信息的，功能与 keepalived 对 LVS 的配置管理功能是完全一样的。ipvsadm 之与 LVS 的关系，就像 iptables 之与 netfilter 的关系。




## 在LVS上新增service
命令格式：
```
ipvsadm -A -t <vip>:<vport> -s <schedule: rr|wrr|lc|wlc|lblc|lblcr|dh|sh|sed|nq>
```
示例：
```sh
# 在 LVS 中为 http 协议添加一个 VIP 为 1.1.1.1 的 service, 并设置调度方式为 Round Robin
$ ipvsadm -A -t 1.1.1.1:80 -s rr
```

## 在LVS上修改存在的Service
命令格式：
```
ipvsadm -E -t <vip>:<vport> -s <schedule> -j <synproxy> =p <timeout> -M <netmask>
-j: <synproxy : disable | enable>
```

示例：
```sh
# 修改 vip 为 1.1.1.1 的 LVS 中 http 服务的调度算法为 Round Robin
$ ipvsadm -E -t 1.1.1.1:80 -s rr
$ ipvsadm -E -f 2 -s rr    # firewall mark type

# 修改 vip 为 2.2.2.2 的 FTP 服务的 TIMEOUT 为 60s，且打开 synproxy
$ ipvsadm -E -t 2.2.2.2:21 -p 60 -j enable
```

注：命令 -E 的选项支持全部指明或部分指明。


## 在LVS上删除一个service
命令格式：
```
ipvsadm -D -t <vip>:<vport>
ipvsadm -D -f fwmark
```
示例：
```sh
# 删除 vip 为 1.1.1.1 的 service
$ ipvsadm -D -t 1.1.1.1:80
```

## 新增RealServer
命令格式：
```
ipvsadm -a -t <vip>:<vport> -r <rip>:<rport> <forward mode>
forward mode : -b(FNAT) | -m(NAT) | -g(DR) | -i(TUNNEL)
```

示例：
```sh
# FULLNAT 模式
$ ipvsadm -a -t 1.1.1.1:80 -r 192.168.1.2:80 -b -w 1
```

## 修改RealServer
命令格式：
```
ipvsadm -e -t <vip>:<vport> -r <rip>:<rport> <forward> -w weight
```

示例：
```sh
# 修改 RealServer 模式为 DR，权重为 12
$ ipvsadm -e -t 1.1.1.1:80 -r 192.168.0.1:80 -g -w 12
```

## 在LVS上删除一个RealServer
命令格式：
```
ipvsadm -d -t <vip>:<vport> -r <rip>:<rport>
```

示例：
```sh
# 删除 vip 为 1.1.1.1 对应的 RealServer 192.168.1.1
$ ipvsadm -d -t 1.1.1.1:80 -r 192.168.1.1:80
```


## 新增LocalAddress
命令格式：
```
ipvsadm -P -t <vip>:<vport> -z <localAddress>
```
示例：
```sh
# 为 vip 1.1.1.1 的 LVS 添加一个 192.168.1.1 的 localAddress
$ ipvsadm -P -t 1.1.1.1:80 -z 192.168.1.2
```

## 删除LocalAddress
命令格式：
```
ipvsadm -Q -t <vip>:<vport> -z <LocalAddress>
```
示例：
```sh
# 删除 vip 为 1.1.1.1 的 LVS 对应的 IP 为 192.168.1.2 的 LocalAddress
$ ipvsadm -Q -t 1.1.1.1:80 -z 192.168.1.2
```

## 查看LocalAddress
命令格式：
```
ipvsadm -G -t <vip>:<vport>
ipvsadm -G
```

示例：
```sh
# 查看所有的 VIP 对应的 LocalAddress
$ ipvsadm -G
# 查看所有的 VIP 为 1.1.1.1 的 LVS 对应的 LocalAddress
$ ipvsadm -G -t 1.1.1.1:80
```


## 查看所有LVS对应RealServer
示例：
```sh
# 查看 LVS 以及对应的 RealServer (不解析 IP 和 PORT)
$ ipvsadm -ln
```

## 清空所有Service
示例：
```sh
$ ipvsadm -C
```


## 二、ipvsadm命令参数详解

**1、用法**

```sh
Usage:
  ipvsadm -A|E -t|u|f service-address [-s scheduler] [-j eanble/disable] [-p [timeout]] [-M netmask]
  ipvsadm -D -t|u|f service-address
  ipvsadm -C
  ipvsadm -R
  ipvsadm -S [-n]
  ipvsadm -P|Q -t|u|f service-address -z local-address
  ipvsadm -G -t|u|f service-address 
  ipvsadm -a|e -t|u|f service-address -r server-address [options]
  ipvsadm -d -t|u|f service-address -r server-address
  ipvsadm -L|l [options]
  ipvsadm -Z [-t|u|f service-address]
  ipvsadm --set tcp tcpfin udp
  ipvsadm --start-daemon state [--mcast-interface interface] [--syncid sid]
  ipvsadm --stop-daemon state
  ipvsadm -h
```


**2、命令**

命令格式支持长选项和短选项的格式：
```sh
--add-service     -A        在内核的虚拟服务器表中添加一条新的虚拟服务器记录
--edit-service    -E        编辑内核虚拟服务器表中的一条虚拟服务器记录
--delete-service  -D        删除内核虚拟服务器表中的一条虚拟服务器记录
--clear           -C        清除内核虚拟服务器表中的所有记录
--save            -S        保存虚拟服务器规则，输出为-R 选项可读的格式
--restore         -R        恢复虚拟服务器规则(标准输入读取)
--add-laddr       -P        为 Service 添加 LocalAddress (仅 FULLNAT)
--del-laddr       -Q        为 Service 删除 LocalAddress (仅 FULLNAT)
--get-laddr       -G        查看 Service 的 LocalAddress (仅 FULLNAT)
--add-server      -a        在内核虚拟服务器表的一条记录里添加一条新的真实服务器记录
--edit-server     -e        编辑一条虚拟服务器记录中的某条真实服务器记录
--delete-server   -d        删除一条虚拟服务器记录中的某条真实服务器记录
--list            -L|-l     显示内核虚拟服务器表
--zero            -Z        虚拟服务表计数器清零，清空当前的连接数量等
--set tcp tcpfin udp        设置连接超时值
--start-daemon              启动同步守护进程，可以是 master 或 backup
--stop-daemon               停止同步守护进程
```


**3、选项**

```sh
--tcp-service  -t vip:vport     说明虚拟服务器提供的是 tcp 的服务
--udp-service  -u vip:vport     说明虚拟服务器提供的是 udp 的服务
--fwmark-service  -f fwmark     说明是经过iptables 标记过的服务类型
--scheduler    -s scheduler     one of rr|wrr|lc|wlc|lblc|lblcr|dh|sh|sed|nq,
                                默认调度方式是：wlc.
--persistent   -p [timeout]     来自同一个客户的多次请求，将被同一台真实的服务器处理
                                默认：300s
--netmask      -M netmask       persistent granularity mask
--real-server  -r rip:rport     真实的服务器
--gatewaying   -g               指定LVS 的工作模式为直接路由模式
--ipip         -i               指定LVS 的工作模式为隧道模式
--fullnat      -b               指定LVS 的工作模式为 FULLNAT 模式
--masquerading -m               指定LVS 的工作模式为NAT 模式
--weight       -w weight        真实服务器的权值
--u-threshold  -x uthreshold    upper threshold of connections
--l-threshold  -y lthreshold    lower threshold of connections
--mcast-interface interface     指定组播的同步接口
--syncid sid                    syncid for connection sync (default=255)
--connection   -c               显示 LVS 目前的连接
--timeout                       显示 tcp tcpfin udp 的 timeout 值
--daemon                        显示同步守护进程状态
--stats                         显示统计信息
--rate                          显示速率信息
--thresholds                    output of thresholds information
--persistent-conn               output of persistent connection info
--sort                          对虚拟服务器和真实服务器排序输出
--numeric      -n               输出IP 地址和端口的数字形式
```

