---
layout: post
title:  "sed入门教程系列之：分支和测试"
categories: Linux工具
tags: sed
author: 肖邦
---

* content
{:toc}

> sed 是一个比较古老的，功能十分强大的用于文本处理的流编辑器，加上正则表达式的支持，可以进行大量的复杂的文本编辑操作。sed 本身是一个非常复杂的工具，有专门的书籍讲解 sed 的具体用法，但是个人觉得没有必要去学习它的每个细节，那样没有特别大的实际意义。网上也有很多关于 sed 的教程，我也是以学以致用的心态来学习 sed 的常见的用法，并进行系统的总结，内容基本覆盖了 sed 的大部分的知识点。文中的内容比较简练，加以实际示例来帮助去理解 sed 的使用。




本系列文章目录：
* [sed 入门教程系列 01：基本原理介绍](/articles/sed-tutorial-series-01.html)
* [sed 入门教程系列 02：基本正则表达式](/articles/sed-tutorial-series-02.html)
* [sed 入门教程系列 03：扩展正则表达式](/articles/sed-tutorial-series-03html)
* [sed 入门教程系列 04：sed语法和常用选项](/articles/sed-tutorial-series-04.html)
* [sed 入门教程系列 05：数字定址和正则定址](/articles/sed-tutorial-series-05.html)
* [sed 入门教程系列 06：基本子命令](/articles/sed-tutorial-series-06.html)
* [sed 入门教程系列 07：sed工作模式](/articles/sed-tutorial-series-07.html)
* [sed 入门教程系列 08：高级子命令](/articles/sed-tutorial-series-08.html)
* **sed 入门教程系列 09：分支和测试**
* [sed 入门教程系列 10：sed实战练习](/articles/sed-tutorial-series-10.html)


## 分支和测试
分支命令用于无条件转移，测试命令用于有条件转移。

**1、分支branch**

* 跳转的位置与标签相关联。
* 如果有标签则跳转到标签所在的后面行继续执行。
* 如果没有标签则跳转到脚本的结尾处。
* 标签：以冒号开始后接标签名，不要在标签名前后使用空格。

**2、跳转到标签指定位置**

测试文件：
```bash
$ grep seker /etc/passwd
seker:x:500:500::/home/seker:/bin/bash
```
例子1：
```bash
$ grep seker /etc/passwd | \
  sed ':top;s/seker/blues/;/seker/b top;s/5/555/'
# 结果：blues:x:55500:500::/home/blues:/bin/bash
```

例子2：
```bash
# 选择执行
$ grep 'seker' /etc/passwd | \
  sed 's/seker/blues/;/seker/b end;s/5/555/;:end;s/5/666/'
#结果：blues:x:66600:500::/home/seker:/bin/bash
```
例子3：
```bash
# 测试命令，如果前一个替换命令执行成功则跳转到脚本末尾(case结构)
$ grep 'seker' /etc/passwd | \
  sed 's/seker/ABC;t;s/home/DEF/;t;s/bash/XYZ/'
# 结果：ABC:x:500:500::/home/seker:/bin/bash
```
例子4：
```bash
$ grep 'zorro' /etc/passwd | \
  sed 's/seker/ABC/;t;s/home/DEF/;t;s/bash/XYZ'
# 结果：zorro:x:500:500::/DEF/zorro:/bin/bash
```

例子5：
```bash
# 与标签关联，跳转到标签位置。
$ grep 'seker' /etc/passwd | \
  sed 's/seker/ABC/;t end;s/home/DEF/;t;end;s/bash/XYZ'
# 结果：ABC:x:500:500::/home/seker:/bin/XYZ
```
