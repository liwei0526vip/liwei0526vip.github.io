---
layout: post
title:  "CentOS7 网卡命名为 eth0 格式"
categories: Linux系统
tags:  CentOS7 网卡命名
author: 肖邦
---

* content
{:toc}

> Linux 操作系统的网卡设备的传统命名方式是 eth0、eth1、eth2 等，而 CentOS7 提供了不同的命名规则，默认是基于固件、拓扑、位置信息来分配。这样做的优点是命名全自动的、可预知的，缺点是比 eth0、wlan0 更难读，比如 ens33 。




## 命名规则策略
* **规则 1**。对于板载设备命名合并固件或 BIOS 提供的索引号，如果来自固件或 BIOS 的信息可读就命名，比如eno1，这种命名是比较常见的，否则使用规则 2。

* **规则 2**。命名合并固件或 BIOS 提供的 PCI-E 热插拔口索引号，比如 ens1，如果信息可读就使用，否则使用规则3。

* **规则 3**。命名合并硬件接口的物理位置，比如 enp2s0，可用就命名，失败直接到方案5。

* **规则 4**。命名合并接口的 MAC 地址，比如 enx78e7d1ea46da，默认不使用，除非用户选择使用此方案。

* **规则 5**。使用传统的方案，如果所有的方案都失败，使用类似 eth0 这样的样式。


## 网卡名称字符含义
1、**前2个字符的含义**
```sh
"en"     以太网　　　　Ethernet
"wl"     无线局域网　　WLAN
"ww"     无线广域网　　WWAN
```

2、**第3个字符根据设备类型选择**
```
o<index>:           on-board device index number
s<slot>:            hotplug slot index number
x<MAC>:             MAC address
p<bus>s<slot>:      PCI geographical location
p<bus>s<slot>:      USB port number chain
``` 

## 修改网卡名称样式为ethx
如果不习惯使用新的命名规则，可以恢复使用传统的方式命名，编辑 grub 文件，增加两个变量，再使用 grub2-mkconfig 重新生成 grub 配置文件即可。

1、**编辑 grub 配置文件**
```sh
$ vim /etc/sysconfig/grub
# 其实是/etc/default/grub的软连接

# 为GRUB_CMDLINE_LINUX变量增加2个参数：
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=cl/root \
 rd.lvm.lv=cl/swap net.ifnames=0 biosdevname=0 rhgb quiet"
```

2、**重新生成 grub 配置文件**
```sh
$ grub2-mkconfig -o /boot/grub2/grub.cfg
```
然后重新启动 Linux 操作系统，通过 ip addr 可以看到网卡名称已经变为 eth0 。

3、**修改网卡配置文件**

原来网卡配置文件名称为 ifcfg-ens33，这里需要修改为 ethx 的格式，并适当调整网卡配置文件。
```sh
$ mv /etc/sysconfig/network-scripts/ifcfg-ens33 \
     /etc/sysconfig/network-scripts/ifcfg-eth0
# 修改ifcfg-eth0文件如下内容(其它内容不变)
NAME=eth0
DEVICE=eth0
$ systemctl restart network.service    # 重启网络服务
```

注意：ifcfg-ens33 文件最好删除掉，否则重启 network 服务时候会报错。
