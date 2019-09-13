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

