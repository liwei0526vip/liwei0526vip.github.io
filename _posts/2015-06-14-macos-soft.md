---
layout: post
title:  "MacOS 常用软件安装"
categories: 工具效率
tags: MacOS
author: 肖邦
---

* content
{:toc}

> 记录一下 Mac OS 系统常用的必备软件安装，作为笔记或备忘录。




## MacOS 必备软件清单

* item2 + sshpass
* vscode
* 有道云笔记
* chrome
* ctags
* zsh + oh-my-zsh
* Beyond Compare



## MacOS 常用配置

* **开启触摸板`选择文字`**："设置" -> “辅助功能” -> "鼠标与触摸板" -> "触控板选项" -> 选择"启用拖移"，并选择"使用拖移锁定" -> “好”
* **关闭仪表盘桌面**："设置" -> "调度中心" -> "仪表盘【关闭】"


## MacOS 常用快捷键

* 删除文件：`Command + Delete`




## brew 安装

* 安装
  ```bash
  # 参考链接: https://brew.sh
  $ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
  # 安装过程需要几分钟，需要耐心等待
  ```

## Chrome 常用插件

* Markdown Preview Plus
* 眼睛卫士
* Octotree
* Vimium
* OneTab
* Gliffy Diagrams


## sshpass 安装

* 安装
  ```bash
  $ brew install https://raw.githubusercontent.com/kadwanev/bigboybrew/master/Library/Formula/sshpass.rb
  ```

## zsh 替代 bash

* MacOS 上切换到 zsh
  ```bash
  $ chsh -s /bin/zsh
  
  # 如果Linux系统没有zsh，则可以执行如下命令安装
  $ yum install zsh -y
  ```
* 安装 oh-my-zsh 主题
  ```bash
  $ git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
  $ cp ~/.zshrc ~/.zshrc.orig  # 备份（如果存在的话）
  $ cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
  # 重启终端，生效
  ```
* 选择主题
  ```bash
  # 主题所在目录:
  ~/.oh-my-zsh/themes
  
  $ vim ~/.zshrc
  ZSH_THEME="xiong-chiamiov-plus"  # 打开 ~/.zshrc 添加一行
  ```

## ctags 实现函数跳转

* 替代 MacOS 自带 ctags
  ```bash
  $ brew install ctags
  $ echo 'alias ctags="`brew --prefix`/bin/ctags"' >> $HOME/.zshrc
  $ exec $SHELL

  # ctags -R --exclude=.git --exclude=log *
  ```

## 实现 vimIDE 功能

* 安装
  ```bash
  $ git clone https://github.com/liwei0526vip/vimide.git
  $ mv vimide/vim ~/.vim
  $ mv vimide/vimrc ~/.vimrc
  # 在vim中复制文本：先按着 Shift 键，然后鼠标选中要赋值的文本进行复制
  ```

## Beyond Compare 安装

* 下载安装

  ```bash
  https://www.scootersoftware.com/download.php
  ```

* 破解（启动前删除注册证书）

  ```bash
  $ cd /Applications/Beyond Compare.app/Contents/MacOS
  $ mv BCompare BCompare.real
  
  # 新建文件BCompare，内容如下：
  #!/bin/bash
  rm "/Users/$(whoami)/Library/Application Support/Beyond Compare/registry.dat"
  "`dirname "$0"`"/BCompare.real $@
  # 赋予可执行权限
  $ chmod a+x BCompare
  ```

* 重新启动应用程序即可。

## 如何批量下载网页资源

* 使用 wget 进行下载

  ```bash
  $ wget -r -nd -np --accept=pdf http://fast.dpdk.org/doc/pdf-guides/
  # --accept 选项指定资源类型格式 pdf
  ```


## virtualbox 虚拟机网络配置

> 虚拟机网络设置最简单的方式就是使用 `桥接模式`，所有的网络场景都能连通。但也存在几个缺点：
> 1. DHCP 的话，IP 地址不固定
> 2. 手动配置 IP 的话，IP 地址有时候是稀缺资源
> 3. 如果笔记本换工作环境的话，网段信息有可能发生变化，需要重新适配新网络

- 目前采用的解决方式：`NAT 网络` + `Host Only`
- 先了解几种网络的应用场景

|                    | NAT  | 桥接 | Host Only      |
| ------------------ | ---- | ---- | -------------- |
| 虚拟机 -> 主机     | 1    | 1    | 默认不能需设置 |
| 主机 -> 虚拟机     | 0    | 1    | 默认不能需设置 |
| 虚拟机 -> 其它主机 | 1    | 1    | 默认不能需设置 |
| 其它主机 -> 虚拟机 | 0    | 1    | 默认不能需设置 |
| 虚拟机之间         | 0    | 1    | 1              |

- 几个步骤
  - 全局设置 Nat 网络：选择 "管理" -> "全局设定" -> "网络" -> "添加 Nat 网络" -> "确定"
  - 添加主机网络器：选择 "管理" -> "主机网络管理器" -> "新建主机"，选手动配置网卡即可
  - 设置虚拟机网络："对应虚拟机" -> "设置" -> "网络" -> "网卡1设置（选择Nat网络）" -> "网卡2（选择Host Only网络）"
  - 进入虚拟机设置：
    - 网卡1：默认 dhcp 方式，不用管；
    - 网卡2：设置 Host Only 网卡 IP 地址，设置静态地址和掩码即可，无需设置网关，网段设置与步骤 2 中网段一致即可；
    - 网卡2：一般网段信息是 `192.168.56.0`
  - 重启虚拟机网络服务
    - 从宿主机 ping 网卡 2 配置的 IP 地址，`ping 192.168.56.200`
    - 在虚拟机中，ping 外网，测试公网是否连通
- 到此为止，vbox 设置完毕

## Git 命令行常用快捷键

> 需要安装 oh-my-zsh

- `gst` - `git status`
- `gco` - `git checkout`
- `gcmsg` - `git commit -m`
- `gb` - `git branch`

