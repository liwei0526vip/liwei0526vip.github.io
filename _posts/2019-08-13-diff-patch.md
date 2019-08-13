---
layout: post
date: "2019-08-13 22:28:15"
title:  "Linux通过diff/patch打补丁"
categories: Linux工具
tags: diff patch
author: 肖邦
---

* content
{:toc}

> 通过 diff 工具生成补丁文件，patch 工具打上补丁。




## 为单个文件生成补丁

```bash
$ diff -up linux-2.6.28.8/net/sunrpc/svc.orig.c linux-2.6.28.8/net/sunrpc/svc.c > patch
```
这条命令会产生类似如下的输出, 你将它重定向到一个文件中, 这个文件就是 patch
```diff
diff -up linux-2.6.28.8/net/sunrpc/svc.orig.c 2009-03-17 08:50:04.000000000 +0800
+++ linux-2.6.28.8/net/sunrpc/svc.c 2009-03-30 19:18:41.859375000 +0800
@@ -1050,11 +1050,11 @@ svc_process(struct svc_rqst *rqstp)
```
参数详解:
* `-u` 显示有差异行的前后几行(上下文), 默认是前后各3行, 这样, patch 中带有更多的信息.
* `-p` 显示代码所在的c函数的信息.


## 为多个文件生成补丁

```bash
$ diff -uprN linux-2.6.28.8.orig/net/sunrpc/ linux-2.6.28.8/net/sunrpc/ > patch
```

参数详解:
* `-r` 递归地对比一个目录和它的所有子目录(即整个目录树).
* `-N` 如果某个文件缺少了, 就当作是空文件来对比. 如果不使用本选项, 当diff发现旧代码或者新代码缺少文件时, 只简单的提示缺少文件. 如果使用本选项, 会将新添加的文件全新打印出来作为新增的部分


## 打补丁

生成的补丁中, 路径信息包含了你的Linux源码根目录的名称, 但其他人的源码根目录可能是其它名字, 所以, 打补丁时, 要进入你的Linux源码根目录, 并且告诉patch工具, 请忽略补丁中的路径的第一级目录(参数-p1)

```bash
$ patch -p1 < patch1.diff
```


## 简单示例

```bash
$ diff -uparN linux-2.6.31.3 linux-2.6.31.3_1/ > mypatch
$ cd linux-2.6.31.3
$ patch -p1 < mypatch
```

