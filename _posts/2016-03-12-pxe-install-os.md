---
layout: post
title:  "通过PXE无人值守安装"
categories: Linux工具
tags: pxe
author: 肖邦
---

* content
{:toc}




> 通过传统的方式安装和部署计算机时，都需要人工干预的方式完成安装。如果需要部署大量的类似功能的工作站或服务器，则需要耗费大量的时间。同时传统的安装方式，每台计算机都需要光驱设备及安装光盘等介质，会额外增加部署成本。因此，许多系统管理员都希望能够通过一种网络化的无人值守的自动安装方式将操作系统部署到目标计算机中。

## 一、相关服务和工具

**1、PXE协议**

PXE 是由 Intel 设计的协议，计算机可以通过 PXE 协议从网络引导启动。PXE 协议在启动过程分为 client 和 server 端，PXE 协议运行过程主要解决两个问题：首先解决 IP 地址的问题，然后解决如何传输操作系统启动文件和安装文件的问题。对于第一个问题，可以通过 DHCP Server 解决，通常情况下 DHCP 服务器主要用于分配 IP 地址给客户端。但在 PXE 环境下，DHCP 服务器需要额外加载 PXE 的相关配置。针对第二个问题，在启动初期因为 PXE client 中有相应的 TFTP 客户端，可以通过 TFTP 协议到 TFTP 服务器中下载相关文件启动计算机。后期在安装过程中，则通过 FTP 或 NFS 协议提供大量的操作系统安装文件的下载。

**2、Kickstart**

通过传统的方式安装和部署计算机时，都会要求通过交互的方式，回答各类问题，以完成安装和部署任务，过程繁琐，且无法实现自动化。红帽公司开发了 Kickstart 的安装方法，通过 ks 文件可以解决所有普通安装方式中需要回答的问题。可以通过 system-config-kickstart 工具定制 ks 文件，也可以通过相关语法来手工编写安装脚本。

**3、CentOS操作系统**

本次实验中所使用和安装的操作系统为CentOS 7，理论上 CentOS 6也是适用的。

**4、DHCP**

动态主机配置协议，主要用于给 DHCP 客户端自动分配 IP 地址，便于用户管理网络内部的计算机。针对 PXE 环境下，DHCP 服务器除分配 IP 地址外，还需要额外配置"next-server"选项定义 TFTP 服务器的地址，设置"filename"选项定义启动文件的名称。并且启动"booting"与"bootp"的支持。

**5、TFTP与FTP**

简单文件传输协议(TFTP)主要用于为客户机与服务器之间进行简单文件传输的协议。在 PXE 早期启动过程中，主要通过 TFTP 协议传输"pxelinux.0"。文件传输协议(FTP)，适用于大量文件传输的情形，在后期安装过程，主要通过 FTP 协议传输 Linux 操作系统的安装包。

 

## 二、安装配置FTP服务

FTP 服务主要是下载 ks.cfg 文件和 操作系统文件的，也可以用 HTTP 或 NFS 来代替。

**1、安装vsftpd服务**

```bash
[root@localhost ~]$ yum install -y vsftpd
```

**2、提供操作系统镜像文件**

FTP 默认配置即可，我们需要适用匿名用户。通过ftp安装操作系统，我们需要把操作系统镜像文件拷贝到这个匿名用户目录
```bash
[root@localhost ~]$ mount /dev/cdrom /var/ftp/pub/    # /var/ftp/pub是ftp的匿名用户目录
```

**3、启动ftp服务**
```bash
[root@localhost ~]$ systemctl start  vsftpd    # 启动ftp服务
[root@localhost ~]$ systemctl enable vsftpd    # 设置开机启动
```

## 三、安装dhcp和tftp

DHCP 和 TFTP 服务可以选择单独分别去安装，也可以通过安装 dnsmasq 服务，来实现 DHCP 和 TFTP 的功能。

**1、安装dnsmasq软件包**

```bash
[root@localhost ~]$ yum install -y dnsmasq
```

**2、配置dnsmasq**

dnsmasq 的配置文件是 /etc/dnsmasq.conf，主要是去掉以下相关的注释，并设置修改 DHCP 的范围和 TFTP 的根目录。
```
bogus-priv
filterwin2k
interface=eth0
dhcp-range=192.168.0.50,192.168.0.100,12h
dhcp-boot=pxelinux.0
enable-tftp
tftp-root=/var/tftp    # tftp目录默认是没有的，需要手动创建
dhcp-authoritative
```

**3、创建tftp根目录**

```bash
[root@localhost ~]$ mkdir /var/tftp
```

**4、启动dnsmasq**

```bash
[root@localhost ~]$ systemctl start  dnsmasq
[root@localhost ~]$ systemctl enable dnsmasq
``` 

## 四、拷贝和准备相关文件

**1、从iso中拷贝内核镜像和文件系统镜像**

```bash
cp /var/ftp/pub/images/pxeboot/initrd.img /var/tftp/    # 拷贝文件系统镜像
cp /var/ftp/pub/images/pxeboot/vmlinuz    /var/tftp/    # 拷贝内核镜像文件
```

**2、生成pxe启动文件pxelinux.0**

```bash
yum install -y syslinux                           # 安装pxelinux.0所需要的包
rpm -ql syslinux | grep "pxelinux.0"              # 查询文件所在目录
cp /usr/share/syslinux/pxelinux.0 /var/tftp/      # 拷贝pxelinux.0文件到tftp根目录
```

**3、准备默认的菜单配置文件**

```bash
mkdir /var/tftp/pxelinux.cfg/         # 创建pxelinux.cfg目录，固定目录名称
vim /var/tftp/pxelinux.cfg/default    # default文件，必须为这个名称
# 编辑内容如下
default linux
prompt 1
timeout 60
display boot.msg
label linux
  kernel vmlinuz
  append initrd=initrd.img text ks=ftp://192.168.0.3/ks.cfg    # 这个地方指定了ks.cfg文件下载路径，后边会生成该文件
```

**4、生成kickstart文件**

kickstart 文件可以通过 system-config-kickstart 可视化工具来进行配置，生成 ks.cfg 文件；也可以通过已经安装好的操作系统的模板文件 anaconda-ks.cfg 来稍加修改即可。下边的 ks.cfg 文件是做实验时的样本，内容如下(加粗为修改部分)：
```
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use CDROM installation media
install
url --url=ftp://192.168.0.3/pub/    # 需要指定安装方式通过ftp来下载安装操作系统
# Use graphical install
graphical
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network  --bootproto=dhcp --device=ens33 --onboot=off --ipv6=auto --no-activate
network  --hostname=localhost.localdomain

# Root password
rootpw --iscrypted $6$LK7yftVlSa2zcGia$4loHYYWZUosdWvZA7Qzf.0lhmrcD5n26BK1xWm7QCNBdbBSjC7MK7yAYRvmIXGI8wu.t96jo6m8RRmNyjsKY60
# System services
services --disabled="chronyd"
# System timezone
timezone Asia/Shanghai --isUtc --nontp
# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
autopart --type=lvm
# Partition clearing information
clearpart --all --initlabel --drives=sda
... ... ... ...    # 还有很多内容
```
拷贝 ks.cfg 文件到 FTP 目录
```bash
[root@localhost ~]$ cp /root/anaconda-ks.cfg /var/ftp/ks.cfg
[root@localhost ~]$ chmod +r /var/ftp/ks.cfg
```

## 五、客户端安装操作系统验证

以上工作完成之后，就可以开始安装操作系统了：

* 1、准备一台适当配置的物理机
* 2、连接网线，与服务器在同一个局域网内
* 3、设置 BIOS 从网卡启动
* 4、等待安装......

遇到的问题，有的主机即使设置了 BIOS 从 network 启动，仍然不能正常从网络来启动安装，需要仔细查找到 BISO 的关于 PXE 的开关设置，然后将其打开，每个主机的 BIOS 设置方式都不同，需要自己根据具体的硬件来设置。
