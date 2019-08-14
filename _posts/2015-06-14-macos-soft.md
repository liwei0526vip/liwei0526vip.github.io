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

> * item2 + sshpass
> * vscode
> * 有道云笔记
> * chrome
> * ctags
> * zsh + oh-my-zsh


## brew 安装

```bash
# 参考链接: https://brew.sh
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
# 安装过程需要几分钟，需要耐心等待
```


## sshpass 安装

```bash
$ brew install https://raw.githubusercontent.com/kadwanev/bigboybrew/master/Library/Formula/sshpass.rb
```

## zsh 替代 bash

* MacOS 上切换到 zsh
  ```bash
  $ chsh -s /bin/zsh
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
  # 主题所在目录
  ~/.oh-my-zsh/themes
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

```bash
$ git clone https://github.com/liwei0526vip/vimide.git
$ mv vimide/vim ~/.vim
$ mv vimide/vimrc ~/.vimrc
# 在vim中复制文本：先按着 Shift 键，然后鼠标选中要赋值的文本进行复制
```
