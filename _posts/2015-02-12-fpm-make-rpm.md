---
layout: post
title:  "使用fpm工具打包RPM" 
categories: Linux工具
tags: fpm rpm
author: 肖邦
---

* content
{:toc}

> 文档记录使用 fpm 工具为 lvs-AutoOps 制作 RPM 包的过程。包括所依赖的工具 conf2json 和 check_keepalived 的 RPM 包制作过程。




## 安装部署fpm工具

fpm 使用 ruby 写的一个快速构建安装包的工具，使用 fpm 工具可以省去了写 .spec 文件，使用很简单。

* 安装ruby环境

  ```bash
  $ yum install rubygems ruby-devel rubugems-devel -y
  $ yum install gcc rpm-build -y
  ```
* 更换ruby国内源

  ```bash
  $ gem sources --add http://gems.ruby-china.com/
  $ gem sources --remove http://rubygems.org/
  ```
* 安装打包工具fpm

  ```bash
  $ gem install json -v 1.8.3 -V
  $ gem install ffi -v 1.9.10 -V
  $ gem install fpm -v 1.4.0 -V
  ```


## 打包check_keepalived工具

* 准备环境目录

  ```bash
  $ mkdir /root/fpm_install -p
  $ mkdir /root/rpms/x86_64
  $ cd /root/fpm_install
  $ mkdir usr/local/sbin -p
  $ cp -p /root/conf2json usr/local/sbin/
  $ cp -p /root/check_keepalived usr/local/sbin
  $ chmod +x usr/local/sbin/*
  ```

* 执行打包

  ```bash
  $ fpm -s dir -t rpm --vendor CentOS6 -n check_keepalived \
      -v 1.2.2 --iteration 1.el6 \
      --category 'Development/Languages' \
      -m 'liwei42' --url 'www.sina.com.cn' \
      --description 'keepalived syntax check' \
      --license 'GNU' -C /root/fpm_install \
      -p /root/rpms/x86_64 -f usr
  ```

* 参数解释

  ```
  -s dir：输入源类型为目录
  -t rpm：输出类型为rpm
  -n xxx：包名称
  -v xxx：版本号
  --iteration：输出包的迭代号
  -m xxx：打包的作者
  -C xxx：打包的根目录，fpm从此目录开始搜索指定内容
  -p xxx：输出rpm包路径
  -f 选项：如果输出文件已经存在则忽略
  usr ：如果/root/fpm_install下有多个目录，指定只打包usr目录
  ```

* 输出文件

  ```bash
  $ ls /root/rpms/x86_64/check_keepalived-1.2.2-1.el6.x86_64.rpm 
    /root/rpms/x86_64/check_keepalived-1.2.2-1.el6.x86_64.rpm
  ```
  RPM 包包含的文件列表：
  ```bash
  $ rpm -qpl /root/rpms/x86_64/check_keepalived-1.2.2-1.el6.x86_64.rpm 
    /usr/local/sbin/check_keepalived
    /usr/local/sbin/conf2json
  ```


## 打包lvs-AutoOps程序

* 准备环境目录

  ```bash
  $ cd /root/fpm_install
  $ mkdir data0/loadbalance -p
  $ cd data0/loadbalance
  $ git clone http://git.staff.sina.com.cn/qupeng3/lvs-AutoOps.git
  $ cd lvs-AutoOps
  $ rm resources/__pycache__  .git  -rf
  $ chmod +x lvs-AutoOps/*.sh
  $ cd /root/fpm_install
  ```

* 执行打包

  ```bash
  $ cd /root/fpm_install

  $ fpm -s dir -t rpm --vendor CentOS6 -n lvs-autoops \
      -v 1.1 --iteration 2.el6 \
      --category 'Development/Languages' -m 'liwei42' \
      --url 'www.sina.com.cn' --description 'lvs auto conf tools' \
      --license 'GNU' -C /root/fpm_install \
      -p /root/rpms/x86_64 \
      --after-install data0/loadbalance/lvs-AutoOps/rpm/after_install.sh \
      --before-install data0/loadbalance/lvs-AutoOps/rpm/before_install.sh \
      --after-remove data0/loadbalance/lvs-AutoOps/rpm/after_remove.sh \
      --after-upgrade data0/loadbalance/lvs-AutoOps/rpm/after_upgrade.sh \
      --before-upgrade data0/loadbalance/lvs-AutoOps/rpm/before_upgrade.sh \
      -d 'keepalived_LVS >= 1.2.2-104,check_keepalived >= 1.2.2-1' -f data0
  ```

* 参数解释

  ```
  --after-install xxx.sh：RPM 包安装后会执行的脚本
  --before-install xx.sh：RPM 包安装前会执行的脚本
  --after-remove  xxx.sh：RPM 包删除后会执行的脚本
  --after-upgrade xxx.sh：RPM 包升级后会执行的脚本
  --before-upgrade xx.sh：RPM 包升级前会执行的脚本
  -d 'name >= 1.2.2'：指定依赖的包，包含版本信息
  ```

* 输出文件

  ```bash
  $ rpm -qpl /root/rpms/x86_64/lvs-autoops-1.1-2.el6.x86_64.rpm 
    /data0/loadbalance/lvs-AutoOps/README.md
    /data0/loadbalance/lvs-AutoOps/api_proxy.py
    /data0/loadbalance/lvs-AutoOps/app_proxy.py
    /data0/loadbalance/lvs-AutoOps/lib/api.py
    /data0/loadbalance/lvs-AutoOps/lib/lvs_conf.py
    /data0/loadbalance/lvs-AutoOps/lib/lvs_except.py
    /data0/loadbalance/lvs-AutoOps/lib/lvs_logger.py
    /data0/loadbalance/lvs-AutoOps/lib/lvs_struct.py
    /data0/loadbalance/lvs-AutoOps/main.py
    /data0/loadbalance/lvs-AutoOps/rpm/after_install.sh
    /data0/loadbalance/lvs-AutoOps/rpm/after_remove.sh
    /data0/loadbalance/lvs-AutoOps/rpm/after_upgrade.sh
    /data0/loadbalance/lvs-AutoOps/rpm/before_install.sh
    /data0/loadbalance/lvs-AutoOps/rpm/before_upgrade.sh
    ...... # 没有全部列出文件
    /data0/loadbalance/lvs-AutoOps/使用说明.md

  $ rpm -qp --scripts /root/rpms/x86_64/lvs-autoops-1.1-2.el6.x86_64.rpm
  # 可以列出rpm包中的脚本信息
  ```

* 更新步骤

  ```
  # 1. 更新lvs-AutoOps最新程序
  # 2. 删除多余.git、__pycache__等文件
  # 3. 重新打包
  # 4. 验证rpm
  ```

## 其它事项

* RPM 分为 CentOS5 和 CentOS6，打包时通过 `--iteration 101.el6` 指定
* 发布 RPM 包到公司 yum 源，参考链接：[yum 管理方法 - 知识库](http://10.211.102.50/Proposals/yum%E7%AE%A1%E7%90%86%E6%96%B9%E6%B3%95.html?h=yum)

