---
layout: post
title:  "初步认识DNS系统"
categories: DNS系统
tags: DNS
author: 肖邦
---

* content
{:toc}

> DNS 系统是互联网最重要的基础系统，它为我们的互联网访问带来了极大的便利，可以说现在如果没有 DNS 系统，我们就可能无法进行访问。其实 DNS 系统是整个互联网最大的分布式系统，没有之一。DNS 其实很容易理解，就是将 IP 地址翻译我们容易识记的名称而已。




## 什么是DNS
我们现在每天看新闻、看视频、逛淘宝等访问网站时，如果全部让你用 IP 地址来访问，你肯定不干，哪能记住哪么多 IP 地址呢。所以我们就需要有一张地址薄，能够通过域名查看具体的 IP 地址，它就是我们的经常使用的 DNS。


其实在刚开始计算机数量比较少的时候，当时是通过 `/etc/hosts` 文件来维护计算机名称与 IP 地址的对应关系，每增加一台计算机都需要在 hosts 文件中增加一条记录。当计算机数量呈爆炸式增长时，通过 hosts 文件就很难维护了，比如更新、一致性的问题。这时 DNS 就应运而生了，大家查询和变更数据时都通过统一的 DNS 系统即可，解决了数据同步及不一致的问题。


## DNS的使用场景
DNS 看似功能很简单的系统，在互联网的世界中却是不可或缺的。我来尝试举例说明 DNS 的必要性或者说时功能吧：
* 当你在浏览器中输入 `www.sina.com.cn` 时，试想让你输入 IP 地址来访问新浪门户会是怎么样的感受？
* 在 PC 上我们可以输入 IP 地址，那在手机 APP 上输入 IP 地址也不太现实吧
* 当新浪门户网站的 IP 地址更换时，如果你使用 IP 地址访问的话，新浪如何通知你新变更的 IP 地址是多少？
* 我们可以给一个域名绑定多个 IP 来简单的负载均衡功能。
* 我们可以根据访问 DNS 客户端的真实 IP 地理位置，返回离用户最近的服务器 IP 地址


## 关于DNS服务器
我们日常上网都需要访问 DNS 服务器，可见 DNS 的重要性。同时也是一项重要挑战，一旦 DNS 出现故障，毫不夸张的说整个互联网将会瘫痪。另外，全世界都要访问一台服务器，那么 DNS 服务器的压力非常大，且访问时延非常大，体验很不好。因此 DNS 服务器一定要设计为高可用、高并发和分布式的架构。

DNS 服务器按照功能可以分为如下几类：
* 根 DNS 服务器

  负责返回顶级域名的权威域名服务器的地址，全球共有 13 套根域名服务器(不是 13 台)，大部分在美国，被编号为从 A - M 共 13 个标号。大部分服务器使用 anycast 技术，相同编号的服务器使用同一个 IP 地址，因此 500 多台服务器可以抵抗对其进行的 DDOS 攻击。

* 顶级域 DNS 服务器

  比如，cn域名、com域名、org域名都属于顶级域名，维护这些域名的服务器为顶级域名服务器。

* 权威 DNS 服务器

  直接返回用户请求域名的 IP 地址，比如新浪的权威 DNS: ns1.sina.com.cn

* 本地 DNS 服务器，也叫 LocalDNS

  一般是指所在运营商提供的 DNS，或者像 `8.8.8.8` 或  `114.114.114.114` 这样的公共 DNS


## DNS是如何访问的
我们来分析一下请求 DNS 的整个解析流程：
* 电脑 Chrome 浏览器访问 www.sina.com 网站时，要获取到网站的 IP 地址，就去问 LocalDNS 服务器 `www.sina.com` 的 IP 地址是多少？如果我们是通过 DHCP 配置网络的，那么 LocalDNS 就是运营商比如移动、电信自动分配，它通常就在你网络运营商的某个机房。
* LocalDNS 服务器收到客户端的请求。LocalDNS 服务器上缓存了一张很大的域名与 IP 对应关系表，首先去缓存表去查找，如果找到直接返回给客户端。若找不到的话，LocalDNS 服务器会去请求根域名服务器：老大，请告诉我 `www.sina.com` 的 IP 地址可以么？
* 根域名服务器接收到 LocalDNS 服务器的请求。根域名服务器发现后缀是 .com，说：www.sina.com 这个域名是由 .com 区域管理，我给你它的顶级域名服务器的地址，你问它吧。
* LocalDNS 转向问顶级域名服务器：老二，你能告诉我 www.sina.com 的 IP 地址吗？顶级域名服务器就是负责比如 .com、.net、.org 这些一级域名的管理。.com 顶级域名服务器就告诉 LocalDNS，我给你 www.sina.com 权威 DNS 服务器的地址，你直接去问它吧。
* LocalDNS 又转向问权威 DNS 服务器：请问 www.sina.com 对应的 IP 地址是什么呀？权威 DNS 服务器就是 sina 域名解析结果的原出处，查询后直接将对应的 IP 地址告诉 LocalDNS。
* LocalDNS 终于拿到 www.sina.com 域名的 IP 地址，然后将 IP 地址返回给客户端，并在本地添加了一条缓存记录。
* Chrome 至此获取到 IP 地址，就可以与 sina 服务器直接建立连接，访问网站信息了。


其实浏览器也是有缓存机器的，会缓存 DNS 记录，一定的时间后会过期超时。所以在访问 www.sina.com 时如果浏览器缓存有 sina 的 DNS 记录且没有过期，就会直接使用本地缓存的 IP 地址，这时就不会向 LocalDNS 发起请求了。Chrome 浏览器查看 DNS 缓存的方法：

```bash
# 浏览器地址栏中输入如下信息：
$ chrome://net-internals/#dns
```


## 简单使用DNS
通过上述内容，我们已经大概知道 DNS 的基本工作原理，我们来实际来感受一下 DNS 查询。在 Linux 系统下，可以通过 dig 或 nslookup 工具来进行查询。在 CentOS 系统中 bind-utils 工具包中包含了这两个工具。

我们直接使用新浪的权威 DNS 查询门户网站的域名
```bash
# 查询新浪门户的域名 www.sina.com.cn
$ dig www.sina.com.cn @ns1.sina.com.cn
... ...
;; ANSWER SECTION:
www.sina.com.cn.   3600  IN  CNAME  spool.grid.sinaedge.com.
# 省略了很多内容
```

同样可以指定互联网中的公共 DNS 来查询
```bash
$ dig weibo.cn @1.2.4.8
# weibo.cn 对应的 A 记录
;; ANSWER SECTION:
weibo.cn.               32      IN      A       180.149.139.248
weibo.cn.               32      IN      A       180.149.153.242
weibo.cn.               32      IN      A       180.149.153.187
weibo.cn.               32      IN      A       180.149.153.216

# 新浪的 4 台权威 DNS 服务器域名
;; AUTHORITY SECTION:
weibo.cn.               14117   IN      NS      ns3.sina.com.cn.
weibo.cn.               14117   IN      NS      ns1.sina.com.cn.
weibo.cn.               14117   IN      NS      ns4.sina.com.cn.
weibo.cn.               14117   IN      NS      ns2.sina.com.cn.

# 新浪 4 台权威 DNS 服务器对应的 IP 地址
;; ADDITIONAL SECTION:
ns1.sina.com.cn.        59877   IN      A       202.106.184.166
ns2.sina.com.cn.        73979   IN      A       61.172.201.254
ns3.sina.com.cn.        73979   IN      A       123.125.29.99
ns4.sina.com.cn.        65585   IN      A       121.14.1.22
```

默认 DNS 使用 UDP 协议进行查询的，也可以指定 TCP 协议
```bash
$ dig www.sina.com.cn +tcp
# 不指定 DNS 的话，默认会按系统 ／etc/resolv.conf 配置的 DNS 来查询
```
