---
layout: post
title:  "TCP的三次握手与四次挥手"
categories: 网络协议
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

TCP四次挥手，也就是应用程序不需要数据通信了，发起断开TCP连接。

![TCP四次挥手](/image/20190623_tcp_termination.png)

* **第一次挥手**。客户端发起 FIN 包（FIN = 1）,客户端进入 `FIN_WAIT_1` 状态。TCP 规定，即使 FIN 包不携带数据，也要消耗一个序号。
* **第二次挥手**。服务器端收到 FIN 包，发出确认包 ACK（ack = u + 1），并带上自己的序号 seq=v，服务器端进入了 `CLOSE_WAIT` 状态。这个时候客户端已经没有数据要发送了，不过服务器端有数据发送的话，客户端依然需要接收。客户端接收到服务器端发送的 ACK 后，进入了 `FIN_WAIT_2` 状态。
* **第三次挥手**。服务器端数据发送完毕后，向客户端发送 FIN 包（seq=w ack=u+1），半连接状态下服务器可能又发送了一些数据，假设发送 seq 为 w。服务器此时进入了 `LAST_ACK` 状态。
* **第四次挥手**。客户端收到服务器的 FIN 包后，发出确认包（ACK=1，ack=w+1），此时客户端就进入了 `TIME_WAIT` 状态。注意此时TCP连接还没有释放，必须经过 2*MSL 后，才进入 `CLOSED` 状态。而服务器端收到客户端的确认包 ACK 后就进入了 `CLOSED` 状态，可以看出服务器端结束 TCP 连接的时间要比客户端早一些。


## 常见TCP相关的面试问题

* 如何优雅的回答面试官的“三次握手”的问题？
> **（1）**、第一次握手：客户端给服务端发一个 SYN 报文，并指明客户端的初始化序列号 ISN(c)。此时客户端处于 SYN_SENT 状态。
> **（2）**、第二次握手：服务器收到客户端的 SYN 报文之后，会以自己的 SYN 报文作为应答，并且也是指定了自己的初始化序列号 ISN(s)，同时会把客户端的 ISN + 1 作为 ACK 的值，表示自己已经收到了客户端的 SYN，此时服务器处于 SYN_REVD 的状态。
> **（3）**、第三次握手：客户端收到 SYN 报文之后，会发送一个 ACK 报文，当然，也是一样把服务器的 ISN + 1 作为 ACK 的值，表示已经收到了服务端的 SYN 报文，此时客户端处于 establised 状态。
> **（4）**、服务器收到 ACK 报文之后，也处于 establised 状态，此时，双方以建立起了链接。

* 为什么连接的时候是三次握手，关闭的时候是四次挥手？
> 答：其实在 TCP 握手的时候，接受端发送 SYN + ACK 的包是将一个 ACK 和一个 SYN 合并到一个包中，所以减少了一次包的发送。而因为 TCP 是全双工通信，在主动关闭方发送 FIN 包后，接收端可能还要发送数据，不能立即关闭服务器端到客户端的数据通道，所以也就不能将服务器端的 FIN 包与对客户端的 ACK 包合并发送，只能先确认 ACK，然后等待无需发送数据时再发送 FIN 包，所以四次挥手时必须是四次数据包的交互。

* 为什么`TIME_WAIT`状态需要经过 `2MSL` 才能返回到`CLOSE`状态？
> 答：MSL 指的是最大的报文生存时间。在客户端发送对服务器端的 FIN 的确认包 ACK 后，这个 ACK 包是有可能不可达的，服务器端如果收不到 ACK 的话会重新发送 FIN 包。所以客户端发送 ACK 后需要留出 2xMSL 时间（ACK 到达服务器 + 服务器发送 FIN 重传包）等待确认服务器端确实收到了 ACK 包。也就是说客户端如果等待 2xMSL 时间也没有收到服务器端的重传包FIN，说明可以确认服务器已经收到客户端发送的 ACK。
> 2、避免新旧连接混淆。

* 三次握手的作用？
> **（1）**、**确认双方的接受能力、发送能力是否正常**。第一次握手：客户端发送网络包，服务端收到了。这样服务端就能得出结论：客户端的发送能力、服务端的接收能力是正常的。第二次握手：服务端发包，客户端收到了。这样客户端就能得出结论：服务端的接收、发送能力，客户端的接收、发送能力是正常的。不过此时服务器并不能确认客户端的接收能力是否正常。 第三次握手：客户端发包，服务端收到了。这样服务端就能得出结论：客户端的接收、发送能力正常，服务器自己的发送、接收能力也正常。所以，只有三次握手才能确认双方的接受与发送能力是否正常。
> **（2）**、指定自己的初始化序列号，为后面的可靠传送做准备。
> **（3）**、如果是 https 协议的话，三次握手这个过程，还会进行数字证书的验证以及加密密钥的生成到。

* 为什么 TCP 连接会采用三次握手，若采用二次握手可以吗？
> 参考三次握手的作用。只有三次握手才能确认双方的接收和发送能力是否正常


## 面试问题扩展

* ISN 代表什么？意义何在？ISN 是固定不变的吗？ISN为何要动态随机？
> 1、ISN，发送方的字节数据编号的原点，让对方生成一个合法的接收窗口。
> 2、如果ISN是固定的，攻击者很容易猜出后续的确认号;增加安全性，为了避免被第三方猜测到，从而被第三方伪造的RST报文Reset。因此 ISN 是动态生成的。
> 3、伪造RST报文，需要sequence number 位于对方的合法接收窗口内。 而由于ISN是动态随机的，猜出对方合法接收窗口难度加大。如果 ISN = 0，那么猜出的难度就大大降低。

* 什么是半连接队列？
> 服务器第一次收到客户端的 SYN 之后，就会处于 SYN_RCVD 状态，此时双方还没有完全建立其连接，服务器会把此种状态下请求连接放在一个队列里，我们把这种队列称之为半连接队列。当然还有一个全连接队列，就是已经完成三次握手，建立起连接的就会放在全连接队列中。如果队列满了就有可能会出现丢包现象。

* 三次握手过程中可以携带数据吗？
> 第一次、第二次握手不可以携带数据，而第三次握手是可以携带数据的。为什么这样呢？大家可以想一个问题，假如第一次握手可以携带数据的话，如果有人要恶意攻击服务器，那他每次都在第一次握手中的 SYN 报文中放入大量的数据，因为攻击者根本就不理服务器的接收、发送能力是否正常，然后疯狂着重复发 SYN 报文的话，这会让服务器花费很多时间、内存空间来接收这些报文。也就是说，第一次握手可以放数据的话，其中一个简单的原因就是会让服务器更加容易受到攻击了。而对于第三次的话，此时客户端已经处于 established 状态，也就是说，对于客户端来说，他已经建立起连接了，并且也已经知道服务器的接收、发送能力是正常的了，所以能携带数据页没啥毛病。

* RFC793明确规定，除了第一个握手报文SYN除外，其它所有报文必须将ACK = 1，试分析其设计的合理性？
> TCP作为一个可靠传输协议，其可靠性就是依赖于收到对方的数据，ACK对方，这样对方就可以释放缓存的数据，因为对方确信数据已经被接收到了。但TCP报文是在IP网络上传输，丢包是家常便饭，接收方要抓住一切的机会，把消息告诉发送方。最方便的方式就是，任何我方发送的TCP报文，都要捎带着ACK状态位。


## 参考链接

* [被面试官问到“三次握手，四次挥手”时该怎么回答](http://blog.itpub.net/31561266/viewspace-2564681/)
* [车小胖微信公众号文章](https://mp.weixin.qq.com/s/S1mv8AE_pQz3uHjRGS7tWg)