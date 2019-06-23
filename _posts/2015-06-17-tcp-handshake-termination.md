---
layout: post
title:  "TCP的三次握手与四次挥手"
categories: TCP协议
tags: TCP
author: 肖邦
---

* content
{:toc}

> TCP协议的重要性不言而喻，而TCP协议也是相当复杂的，如果能够真正理解TCP协议的原理，那么对平时软件开发和故障诊断是很有帮助的。其实在面试中最常见的一个问题就是TCP的三次握手和四次挥手，看似简单，如果想在面试的时候有条不紊的说清楚每一个知识点，还是需要下一番功夫的。这篇文章我试图用通俗浅显的文字来说这个问题，也算是自己的一个总结。




## TCP协议格式字段

TCP在OSI模型的第四层，TCP是没有IP信息的。应用程序将数据封装到TCP的Segment中，然后TCP的Segment会封装到IP的Packet中，然后再封装到以太网Ethernet的Frame中，传送给对端后，各层解析自己的协议，然后把数据交给应用程序处理。

![TCP 协议格式](/image/20190623_tcp_header.png)

* `Sequence Number` 是记录包的序号，用来解决包在网络中乱序的问题。
* `Acknowledgement Number` 就是 ACK ，用于确认已经收到了哪些包，用来解决不丢包的问题。
* `Window` 又叫 `Advertised-Window`，也就是著名的滑动窗口，用来解决流控的。
* `TCP Flag` 也就是包的类型，主要是用于操控 TCP 状态机的。

## TCP三次握手

TCP三次握手，其实就是TCP应用在发送数据前，通过TCP协议跟对端协商好连接信息，建立起TCP的连接关系。我们需要知道，TCP连接并非是在通信设备两端之间建立信号隧道，而本质上就是双方各自维护所需的状态信息，以达到TCP连接的效果。所以TCP状态机是TCP的核心内容，学些TCP就是要搞懂这些状态机。

![TCP 三次握手的状态图](/image/20190623_tcp_handshake.png)

* **第一次握手**。建立连接时，客户端发送 SYN 包（SYN=1 ACK=0 seq = x），并进入 `SYN_SENT` 状态，等待服务器确认。
* **第二次握手**。服务器收到 SYN 包，必须确认客户端的 SYN（ack = x + 1），同时服务器也要发送一个 SYN 包（seq = y），即 `SYN+ACK` 包（SYN=1 ACK=1 seq=y, ack=x+1），此时服务器进入 `SYN_RCVD` 状态。
* **第三次握手**。客户端收到服务器的 `SYN+ACK` 包，向服务器发送确认包（ack=y+1），此时发送完毕，客户端进入 ESTABLISHED 状态。等服务器端收到客户端发送的 ACK 包后也进入 ESTABLISHED 状态，完成三次握手。

## TCP四次挥手



