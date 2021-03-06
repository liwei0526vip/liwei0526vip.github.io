---
layout: post
title:  "sed入门教程系列之：sed工作模式"
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
* **sed 入门教程系列 07：sed工作模式**
* [sed 入门教程系列 08：高级子命令](/articles/sed-tutorial-series-08.html)
* [sed 入门教程系列 09：分支和测试](/articles/sed-tutorial-series-09.html)
* [sed 入门教程系列 10：sed实战练习](/articles/sed-tutorial-series-10.html)


## sed工作模式

**1、模式空间和保持空间**

* 模式空间初始化为空，处理完一行后会自动输出到屏幕并清除模式空间；
* 保持空间初始化为一个空行，也就是默认带一个 \n，处理完后不会自动清除。
* 模式空间和保持空间，从程序的角度去看，其实就是 sed 在工作的时候占用了一些内存空间和地址，sed 工作完毕就会把内存释放并归还给操作系统。

**2、sed工作流程**

大概简单描述一下 sed 的工作流程，读取文件的一行，存入模式空间，然后进行所有子命令的处理，处理完后默认会将模式空间的内容输出打印到标准输出，也就是在屏幕上显示出来，接着清空模式空间的内存，继续读取下一行的内容到模式空间，继续处理，依次循环处理。

**3、模式空间和保持空间的置换**
* h：把模式空间内容覆盖到保持空间中
* H：把模式空间内容追加到保持空间中
* g：把保持空间内容覆盖到模式空间中
* G：把保持空间内容追加到模式空间中
* x：交换模式空间与保持空间的内容

**4、实例用法**

测试文件：
```bash
$ cat test.txt
11111
22222
33333
44444
```
例子1：
```bash
$ sed '{1h;2,3H;4G}' test.txt
# 结果：
11111
22222
33333
44444
11111
22222
33333
```

例子2：
```
$ sed '{1h;2x;3g;$G}' test.txt
# 结果：
11111
11111
22222
44444
22222
```
例子3：
```
$ sed '{1!G;h;$!d}' test.txt
# 结果：
44444
33333
22222
11111
```
