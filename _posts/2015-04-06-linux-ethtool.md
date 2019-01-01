---
layout: post
title:  "Linux系统ethtool工具的使用"
categories: Linux工具
tags: ethtool Linux工具
author: 肖邦
---

* content
{:toc}

> ethtool 工具的常见使用方式总结。




## 1、查看网口基本信息
```sh
$ ethtool eth0
```

## 2、点亮网卡灯
```sh
$ ethtool -p eth0 10    # 亮 10 秒
```

## 3、查询网口驱动相关信息
```sh
$ ethtool -i eth0
    driver: igb
    version: 5.3.0-k
    firmware-version: 1.67, 0x80000d38, 18.3.6
    bus-info: 0000:01:00.0
    supports-statistics: yes
    supports-test: yes
    supports-eeprom-access: yes
    supports-register-dump: yes
    supports-priv-flags: no
```

## 4、查询网口收发包统计信息
```sh
$ ethtool -S eth0    # 查看网口所有统计相关信息
$ ethtool -S eth0 | grep rx_queue | grep packets    # 查看网卡各个队列收包数据
    rx_queue_0_packets: 117947
    rx_queue_1_packets: 3050
    rx_queue_2_packets: 1098
    rx_queue_3_packets: 1204
    rx_queue_4_packets: 9679
    rx_queue_5_packets: 1172
    rx_queue_6_packets: 7435
    rx_queue_7_packets: 2403
```

## 5、显示网卡 offload 参数的状态
```sh
$ ethtool -k eth0
    Features for eth0:
    rx-checksumming: on 
    tx-checksumming: on 
    tcp-segmentation-offload: on
    udp-fragmentation-offload: off [fixed]
    generic-segmentation-offload: on
    generic-receive-offload: off
    large-receive-offload: off [fixed]
    ... ...
    ntuple-filters: off [fixed]
    receive-hashing: on
    ... ...
```

参数解释如下：
* `rx-checksumming`: 接收包校验和
* `tx-checksumming`: 发送包校验和
* `tcp-segmentation-offload`: 简称为 TSO，利用网卡对 TCP 数据包分片
* `udp-fragmentation-offload`: 简称为 UFO，针对 UDP 的
* `generic-segmentation-offload`: 简称 GSO，基本思想就是尽可能的推迟数据分片直至发送到网卡驱动之前，检查网卡是否支持分片功能（如 TSO、UFO），如果支持直接发送到网卡，如果不支持就进行分片后再发往网卡。这样大数据包只需走一次协议栈，而不是被分割成几个数据包分别走，这就提高了效率
* `generic-receive-offload`: 简称 GRO，基本思想跟 LRO 类似，克服了 LRO 的一些缺点，更通用。后续的驱动都使用 GRO 的接口，而不是 LRO
* `large-receive-offload`: 简称 LRO，通过将接收到的多个 TCP 数据聚合成一个大的数据包，然后传递给网络协议栈处理，以减少上层协议栈处理 开销，提高系统接收 TCP 数据包的能力
* `ntuple-filters`: ntuple


#### 6、配置网卡 offload 参数
```sh
$ ethtool -K eth0 rx-checksum on|off
$ ethtool -K eth0 tx-checksum-ip-generic on|off
$ ethtool -K eth0 tso on|off
$ ethtool -K eth0 ufo on | off
$ ethtool -K eth0 gso on | off
$ ethtool -K eth0 ntuple on | off
```

#### 7、查看网卡 ntuple 配置规则
```sh
$ ethtool -n eth5
24 RX rings available
Total 480 rules
Filter: 100
        Rule Type: Raw IPv4
        Src IP addr: 0.0.0.0 mask: 255.255.255.255
        Dest IP addr: 10.79.229.11 mask: 255.255.0.0
        TOS: 0x0 mask: 0xff
        Protocol: 0 mask: 0xff
        L4 bytes: 0x0 mask: 0xffffffff
        Action: Direct to queue 11
```
查看现有 flow-hash 配置
```sh
$ ethtool -n eth5 rx-flow-hash tcp4
    TCP over IPV4 flows use these fields for computing Hash flow key:
    IP SA
    IP DA
```

配置 flow-hash
```sh
$ ethtool -N eth5 rx-flow-hash udp4 sd
$ ethtool -N eth5 rx-flow-hash tcp4 sd
$ ethtool -N eth5 rx-flow-hash udp4 sdfn
$ ethtool -N eth5 rx-flow-hash tcp4 sdfn

# 选项含义
s   Hash on the IP source address of the rx packet.
d   Hash on the IP destination address of the rx packet.
f   Hash on bytes 0 and 1 of the Layer 4 header of the rx packet.
n   Hash on bytes 2 and 3 of the Layer 4 header of the rx packet.
```

