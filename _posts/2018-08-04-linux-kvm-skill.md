---
layout: post
title:  "虚拟化 KVM 入门实战详解"
categories: Linux系统
tags:  KVM
author: 肖邦
---

* content
{:toc}

> KVM 是虚拟化技术的一个典型实现，功能非常强大，使用很方便。KVM 本身主要实现对 CPU 的虚拟化，内存和 IO 的虚拟化使用了开源软件 qemu，qemu 是纯软件层面的虚拟化，其实就是个模拟器。KVM 要求 CPU 必须支持硬件虚拟化。




## 0、关于 libvirt 程序
说到 kvm 必须提及 libvirt 程序集，它是用来管理 kvm 虚拟机的，当然其实也可以管理 xen 等其它虚拟化的虚拟机。libvirt 包括三部分：
* `libvirtd` 是后台服务程序；
* `libvirt` 是管理虚拟机的 API 接口，可以通过 python c java 等语言来编写程序管理虚拟机，比较典型的 virt-manager 就是使用 python 写的可视化工具；
* `virsh` 等命令行管理工具。

## 一、环境准备
1、**实验环境**
* 操作系统：CentOS 6.4 x86_64
* 宿主机：vmware workstation 虚拟机

2、**检查宿主机处理器是否支持虚拟化**
```sh
[root@kvm ~]$ egrep -o 'vmx | svm' /proc/cpuinfo | wc -l
```
如果显示数值是 0，则表示该 CPU 不支持虚拟化。

3、**配置或设置宿主机**
* CPU：2-4 core  开启 cpu 虚拟化(bios 设置 或 vmware 设置)
* 内存：4-8 GB
* 硬盘：100 GB

4、**关闭 iptables 和 selinux**

关闭 iptables 服务：
```sh
[root@kvm ~]$ service iptables stop
[root@kvm ~]$ chkconfig iptables off
```
关闭 selinux：
```sh
[root@kvm ~]$ setenforce 0
[root@kvm ~]$ vi /etc/selinux/config
SELINUX=disabled
```

## 二、安装和配置 KVM 环境
1、**安装 kvm 虚拟化相关软件包**
```sh
[root@kvm ~]$ yum install -y kvm virt-* libvirt bridge-utils qemu-img
```

2、**查看 KVM 模块是否加载到内核**
```sh
[root@kvm ~]$ lsmod | grep kvm_intel
kvm_intel              53484  0
kvm                   316506  1 kvm_intel
```
如果没有加载，可以尝试执行命令：`modprobe kvm_intel` ，不行的话，试试重启宿主机。

3、**设置相关网络**
* 设置方式一：网桥模式。
```sh
[root@kvm ~]$ cd /etc/sysconfig/network-scripts/
[root@kvm ~]$ cp ifcfg-eth0 ifcfg-br0
[root@kvm ~]$ vi ifcfg-eth0
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
BRIDGE=br0
```
配置 ifcfg-br0 接口：
```sh
[root@kvm ~]$ vi ifcfg-br0
DEVICE=br0
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=static
IPADDR=172.16.1.8
PREFIX=24
GATEWAY=172.16.1.1
DNS1=114.114.114.114
NAME=br0
[root@kvm ~]$ service network restart
```

* 设置方式二：NAT 模式。
  
  暂略。

4、**启动 libvirtd 相关服务**
  ```sh
[root@kvm ~]$ /etc/init.d/libvirtd start
[root@kvm ~]$ /etc/init.d/messagebus restart
```
遇到错误：
```sh
[root@vip ~]$ /etc/init.d/libvirtd start
libvirtd: relocation error: libvirtd: ... version ... libdevmapper.so.1.02 [失败]
```
解决：
```sh
[root@kvm ~]$ yum upgrade device-mapper-libs
```
结果：
```sh
[root@kvm ~]$ brctl show
bridge name    bridge id           STP enabled    interfaces
br0         8000.000c29181c75   no          eth0
virbr0      8000.525400c207c7   yes         virbr0-nic
```

## 三、安装虚拟机
1、**创建虚拟机镜像**

关于虚拟机镜像，有很多种类型：raw、qcow2、vmdk等，我们推荐使用 qcow2 格式的镜像，因为 qcow2 格式的镜像支持快照，使用的比较广泛。在创建虚拟机之前需要手动去创建 qcow2 格式的镜像磁盘文件，以供安装虚拟机时使用。按照如下命令进行创建：
```sh
$ qemu-img create -f qcow2 -o preallocation=metadata /data/kvm/liwei01.qcow2 50G
```

2、**虚拟机的安装**

* 安装方式 1：通过网络镜像安装，文本控制台，无vnc支持。
```sh
virt-install --name liwei01 --ram 1024 --vcpus 1 \
    -f /data/kvm/liwei01.qcow2  --os-type linux \
    --os-variant rhel6 --network bridge=br0 \
    --graphics none --console pty,target_type=serial \
    --location 'http://mirrors.163.com/centos/6.8/os/i386/' \
    --extra-args 'console=ttyS0,115200n8 serial'
```
* 安装方式 2：通过网络镜像安装，支持 vnc ，默认无文本控制台。
```sh
virt-install --name liwei01 --ram 1024 --vcpus 1 \
    -f /data/kvm/liwei01.qcow2  --os-type linux \
    --os-variant rhel6 --network bridge=br0 \
    --graphics vnc,listen=0.0.0.0,port=5920 \
    --location 'http://mirrors.163.com/centos/6.8/os/i386/'
```
* 安装方式 3：通过 iso 镜像实现本地安装，支持 vnc ，无文本控制台。
```sh
virt-install --name liwei01 --ram 1024 --vcpus 1 \
    -f /data/kvm/liwei01.qcow2  --os-type linux \
    --os-variant rhel6 --network bridge=br0 \
    --cdrom CentOS-6.8-i386-minimal.iso \
    --graphics vnc,listen=0.0.0.0,port=5920
```
* 安装方式 4：通过基础镜像模板快速安装(拷贝)
```sh
# 创建镜像文件
[root@kvm ~]$ qemu-img create -f qcow2 /data/kvm/liwei.qcow2 50G
# 通过 liwei.qcow2 安装虚拟机 ... 安装完毕.
[root@kvm ~]$ cp /data/kvm/liwei.qcow2 /data/kvm/liwei01.qcow2
```
安装命令：
```sh
# 以拷贝的 liwei01.qcow2 为模板进行安装，安装方式是从 liwei01.qcow2 镜像启动
[root@kvm ~]$ virt-install --name liwei01 --ram 1024 --vcpus=1 \
    --disk /data/kvm/liwei01.qcow2,format=qcow2,bus=virtio \
    --network bridge=br0 --graphics vnc,listen=0.0.0.0,port=5904 \
    --boot hd
```
**说明**：本方式创建 img 镜像的时候没有指定 preallocation=metadata 选项，这样存储文件空间显示比较小，方便拷贝，不加这个选项时，在 virt-install 时候需要在 --disk 选项后边加上 bus=virtio，如果不加在安装操作系统的时候似乎是识别不出来磁盘空间，会提示磁盘空间不足。采用这种方式安装的速度非常快，其实就是从已经存在的操作系统镜像启动虚拟机并 define 一个新的虚拟机 liwei01，可以通过脚本快速创建出多个相同配置的虚拟机。当然可以在基础镜像中安装公共的软件包和设置相同的配置，这样后续基于这个 img 安装的虚拟机都有类似的配置，省去重复安装软件包的麻烦。

* 安装方式 5：通过基础镜像模板快速安装(共享)
```sh
# 创建镜像：
[root@kvm ~]$ qemu-img create -f qcow2 -o preallocation=metadata \
              /data/kvm/liwei.qcow2 50G
# 通过 liwei.qcow2 安装虚拟机 ... 安装完毕.
# 以 liwei.qcow2 镜像为模板创建 liwei01.qcow2 镜像
[root@kvm ~]$ qemu-img create -f qcow2 -o backing_file=liwei.qcow2 \
              liwei01.qcow2 10G
```
安装命令：
```sh
[root@kvm ~]$ virt-install --name liwei01 --ram 1024 --vcpus=1 \
    --disk /data/kvm/liwei01.qcow2,format=qcow2,bus=virtio \
    --network bridge=br0 --graphics vnc,listen=0.0.0.0,port=5904 \
    --boot hd
```
**说明**：在创建镜像 liwei01.qcow2 指定了 backing_file=liwei.qcow2 选项，表示以 liwei.qcow2 为后端镜像，以后对虚机 liwei01 的所有的写操作都会记录到 liwei01 镜像，实际操作系统是在 liwei.qcow2 镜像中，liwei.qcow2 镜像是只读的。也就是说后续以 liwei.qcow2 镜像为后端的虚机都共享这个镜像，而具体某个虚机的写操作内容都要记录到对应自己的镜像文件中去。注意和方式 4 的区别。

3、**通过 vnc 或 文本控制台进行系统安装**

* 方式 1 ：通过文本控制台进行管理安装  virsh console liwei01 后续也能用此方式进行登陆管理虚拟机。
* 方式 2：通过 vnc 客户端进行连接，`virsh vncdisplay liwei01 :20` 客户端通过 url: 172.16.1.8:20 进行连接。
* 方式 3：同方式二一样，具体安装过程与普通操作系统安装过程一样，过程略。


## 四、KVM 常用操作
* 开机：`virsh start vm`
* 关机：`virsh shutdown vm`，如果不生效，需要在 vm 中执行：`yum install -y acpid`
* 强关：`virsh destroy vm`
* 删除：`virsh undefine vm`
* 定义：`virsh define vm`
* 挂起：`virsh suspend vm`
* 恢复：`virsh resume vm`
* 虚拟机列表：`virsh list`
* 包含关机的虚机：`virsh list --all`
* 设置自动启动：`virsh autostart vm`
* 关闭自动启动：`virsh autostart --disable vm`
* 登陆虚机控制台：`virsh console vm`，只对指定了 console 的虚机才管用(方式 1)
* 退出虚机控制台：`ctrl + ]`


## 五、虚拟机的克隆
比如：将虚拟机 liwei01 克隆为虚拟机 liwei02

```sh
[root@kvm ~]$ virt-clone --original liwei01 --name liwei02 \
              --file /data/kvm/liwei02.qcow2
```
注意：克隆前需要先关闭虚拟机；克隆完毕，一般需要设置虚拟机的网络。

 
## 六、创建虚拟机的快照
* 创建快照的条件
  - 虚拟机是关机状态。
  - 虚拟机镜像格式是 qcow2。
* 创建快照

  ```[root@kvm ~]# virsh snapshot-create liwei```
* 查看快照列表

  ```[root@kvm ~]# virsh snapshot-list liwei```
* 查看镜像的快照信息

  ```[root@kvm ~]# qemu-img info /data/kvm/liwei.img```
* 切换快照

  ```[root@kvm ~]# virsh snapshot-revert liwei 1477285698```
* 查看当前快照

  ```[root@kvm ~]# virsh snapshot-current liwei```
* 删除快照

  ```[root@kvm ~]# virsh snapshot-delete liwei 1477285698```
* 快照文件存储位置

  ```/var/lib/libvirt/qemu/snapshot```
 
## 七、磁盘扩容和添加磁盘
1、**虚拟机扩容磁盘，给现有磁盘增加容量**
```sh
[root@kvm ~]$ qemu-img resize /data/kvm/liwei.qcow2 +5G
# 重启虚拟机 reboot虚机不生效
[root@kvm ~]$ virsh destroy liwei
[root@kvm ~]$ virsh start   liwei
```
在虚拟机中使用 fdisk -l 查看，通过观察 block 块 id 可以发现存储空间多了，还必须将多余部分分区、格式化使用，默认使用 lvm 。

2、**给虚拟机添加磁盘**

按照如下步骤：
* 关闭虚拟机
* 使用 `qemu-img` 创建磁盘镜像
* 使用 `virsh edit liwei` 编辑虚机配置文件，添加一条磁盘记录，适当修改信息
* 虚拟机开机 -> fdisk -> 格式化 -> ok

注：可以尝试不分区直接格式化，也可以尝试使用 lvm 。


## 八、使用虚拟磁盘恢复虚拟机
思路：首先得有镜像文件(已有) + xml 配置文件
```sh
[root@kvm ~]$  virsh dumpxml liwei > /etc/libvirt/qemu/liwei01.xml
# 编辑配置文件，修改为适当的值
# 添加定义
virsh define /etc/libvirt/qemu/liwei01.xml
virsh list --all   #即可查到该虚拟机
```

## 九、调整CPU、内存规格
如果要调整的 cpu 核数和内存超过安装虚机时指定的最大值，则需要关闭虚机来修改最大值，动态调整的值不能超过设置最大值，但使用值和最大值都是保持一致，一起修改。所以在线动态修改没什么意义，推荐直接修改配置文件就 OK。
```sh
[root@kvm ~]$ virsh edit liwei01
<memory unit='KiB'>1048576</memory>
<currentMemory unit='KiB'>1048576</currentMemory>
<vcpu placement='static'>1</vcpu>
```
重启虚机 liwei01 就ok.


## 十、调整虚拟机网卡
1、**添加虚拟机网卡**
```sh
# 临时命令生效
[root@kvm ~]$ virsh attach-interface liwei --type bridge --source br0  

# 修改虚机配置文件
[root@kvm ~]$ virsh dumpxml liwei > /etc/libvirt/qemu/liwei.xml
[root@kvm ~]$ virsh define /etc/libvirt/qemu/liwei.xml 
```

2、**删除虚拟机网卡**
```sh
$ virsh detach-interface liwei --type bridge --mac 52:54:00:14:86:cf
```

3、**指定网卡类型**

网卡默认类型是 rtl 品牌的网卡，这里设置为 intel 网卡 e1000 系列。修改如下配置文件即可。
```xml
<interface type='bridge'>
  <mac address='52:54:00:b5:68:a4'/>
  <source bridge='br0'/>
  <model type='e1000'/>      # 添加设置字段
  <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```
使用 virsh 重启虚拟机，在虚拟机中查看：
```sh
[root@localhost ~]$ lspci | grep "Ethernet"
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller
```

## 十一、虚拟机的迁移
几个步骤：
* 关闭虚拟机
* 拷贝镜像文件
* 拷贝配置文件
* `virsh define vm`

