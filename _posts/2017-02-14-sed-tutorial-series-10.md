---
layout: post
title:  "sed入门教程系列之：sed实战练习"
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
* [sed 入门教程系列 09：分支和测试](/articles/sed-tutorial-series-09.html)
* **sed 入门教程系列 10：sed实战练习**


## sed实战练习
**实例1：**

删除文件每行的第二个字符。
```bash
$ sed -r 's/(.*)(.)$/\1/'
```

**实例2：**

删除文件每行的最后一个字符。
```bash
$ sed -r 's/(.*)(.)$/\1/'
```

**实例3：**

删除文件每行的倒数第2个单词。
```bash
$ sed -r 's/(.*)([^a-Z]+)([a-Z]+)([^a-Z]+)([a-Z]+)\
  ([^a-Z]*$)/\1\2\4\5/' /etc/passwd
```

**实例4：**

交换每行的第一个字符和第二个字符。
```bash
$ sed -r 's/(.)(.)(.*)/\2\1\3/' /etc/passwd
```

**实例5：**

交换每行的第一个单词和最后一个单词。
```bash
$ sed -r 's/([a-Z]+)([^a-Z]+)(.*)\
  ([^a-Z]+)([a-Z]+)([^a-Z]*$)/\5\2\3\4\1\6/'\
  /etc/passwd
```

**实例6：**

删除一个文件中所有的数字。
```bash
$ sed 's/[0-9]//g' /etc/passwd
```

**实例7：**

用制表符替换文件中出现的所有空格。
```bash
$ sed -r 's/ +/\t/g' /etc/passwd
```

**实例8：**

把所有大写字母用括号 () 括起来。
```bash
$ sed -r 's/([A-Z])/(\1)/g' /etc/passwd
```

**实例9：**

打印每行 3 次。
```bash
$ sed 'p;p' /etc/passwd
```

**实例10：**

隔行删除
```bash
$ sed '0~2{=;d}' /etc/passwd
```

**实例11：**

把文件从第 22 行到第 33 行复制到 56 行后面。
```bash
$ sed '22h;23,33H;56G' /etc/passwd
```

**实例12：**

把文件从第 22 行到第 33 行移动到第 56 行后面。
```bash
$ sed '22{h;d};23,33{H;d};56g' /etc/passwd
```

**实例13：**

只显示每行的第一个单词。
```bash
$ sed -r 's/([a-Z]+)([^a-Z]+)(.*)/\1/' /etc/passwd
```

**实例14：**

打印每行的第一个单词和第三个单词。
```bash
$ sed -r 's/([a-Z]+)([^a-Z]+)([a-Z]+)([^a-Z]+)\
  ([a-Z]+)([^a-Z]+)(.*)/\1\t\5/' /etc/passwd
```

**实例15：**

将格式为 `mm/yy/dd` 的日期格式换成 `mm;yy;dd`
```bash
$ date '+%m/%y/%d' | sed 's/\//;/g'
 ```
