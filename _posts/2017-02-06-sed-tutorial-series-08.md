---
layout: post
title:  "sed入门教程系列之：高级子命令"
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
* **sed 入门教程系列 08：高级子命令**
* [sed 入门教程系列 09：分支和测试](/articles/sed-tutorial-series-09.html)
* [sed 入门教程系列 10：sed实战练习](/articles/sed-tutorial-series-10.html)


## 高级子命令

高级子命令比较少，但是比较复杂，平时用的也会相对少些，却也很重要，有的内容处理不用高级子命令是完成不了的。

* n：读入下一行到模式空间，例：'4{n;d}' 删除第 5 行。
* N：追加下一行到模式空间，再把当前行和下一行同时应用后面的命令。
* P：输出多行模式空间的第一部分，直到第一个嵌入的换行符位置。在执行完脚本的最后一个命令之后，模式空间的内容自动输出。P 命令经常出现在 N 命令之后和 D 命令之前。
* D：删除模式空间中第一个换行符的内容。它不会导致读入新的输入行，相反，它返回到脚本的顶端，将这些指令应用与模式空间剩余的内容。这 3 个命令能建立一个输入、输出循环，用来维护两行模式空间，但是一次只输出一行。

例子1：
```
$ sed 'N;$!P;D' a.txt
#说明：删除文件倒数第二行
```
例子2：
```
$ sed 'N;$!P;$!D;$d' a.txt
# 说明：删除文件最后两行
```
