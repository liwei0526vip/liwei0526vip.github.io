---
layout: post
title:  "网络协议报文格式"
categories: 网络协议
tags: 报文格式
date: "2015-06-20 11:44:30"
author: 肖邦
---

* content
{:toc}




> 简单记录并总结 TCP/IP 协议相关报文字段及含义。

## IP 协议

#### IPv4 协议报文头结构

![IPv4协议报文头结构](/image/IPv4Header.png)

IPv4 协议各字段的含义：
* **版本（Version）**：版本字段表示的是 IP 协议的版本，设置为 4，此字段长 4 位。
* **Internet 头部长度（IHL）**：表示的是 IPv4 头部中包含的 4 字节段的段数。IHL 最大值是 0xF，因此 IPv4 头部的最大长度是 60 字节（15 * 4）。
* **服务类型（Type of Service）**：表示的是数据包希望通过路由器在 IPv4 互联网中发送时能够获得的服务。此字段长 8 bit，RFC 791 初始定义的表示优先级、延迟、吞吐量、可靠性、代价特征等的位。RFC 2474 重新进行了定义，将其定义为有区分服务（DS）字段。RFC 3168 定义，服务类型字段的最后 2bit 用于显式拥塞通告。
* **总长度**：表示的是 IPv4 包的总长度，此字段 16bit，能够表示长度 65535 字节的 IPv4 包。
* **标识符（Identification）**：用于标识当前的 IPv4 包。如果有设备对此 IPv4 包进行了分片，那么所有分片都保持相同的标识 ID，这样目标节点才能将分片进行组合。
* **标记（Flags）**：用于在数据包分片过程中对其进行标记。字段长 3bit，目前只有 2bit 定义，一个用于表示 IPv4 能否进行分片，另一个用于表示这个分片后是否还有其他的分片。
* **分片偏移量（Fragment Offset）**：表示这个分片在相较于 IPv4 有效负载部分所处的位置，字段长 13 bit。

#### IPv6 协议报文头结构

![IPv6 协议报文头结构](/image/IPv46.png)

IPv6 协议各字段的含义：
* 
* 
* 
* 


## TCP 协议

![TCP 协议报文头格式](/image/20190623_tcp_header.png)


## UDP 协议

![UDP 协议报文格式](/image/udpHeader.png)


## ICMP 协议

![ICMP 协议报文格式](/image/icmpHeader.png)


## DNS 协议


## HTTP 协议



> 图片原出处：https://nmap.org/book/tcpip-ref.html
