---
layout: post
title:  "sed入门教程系列之：扩展正则表达式"
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
* **sed 入门教程系列 03：扩展正则表达式**
* [sed 入门教程系列 04：sed语法和常用选项](/articles/sed-tutorial-series-04.html)
* [sed 入门教程系列 05：数字定址和正则定址](/articles/sed-tutorial-series-05.html)
* [sed 入门教程系列 06：基本子命令](/articles/sed-tutorial-series-06.html)
* [sed 入门教程系列 07：sed工作模式](/articles/sed-tutorial-series-07.html)
* [sed 入门教程系列 08：高级子命令](/articles/sed-tutorial-series-08.html)
* [sed 入门教程系列 09：分支和测试](/articles/sed-tutorial-series-09.html)
* [sed 入门教程系列 10：sed实战练习](/articles/sed-tutorial-series-10.html)

## 扩展正则表达式

扩展正则表达式是在基本正则表达式中扩展出来的，内容不是很多，使用频率上可能没有基本正则表达式那么高，但是扩展正则依然很重要，很多情况下没有扩展正则是搞不定的。sed 命令使用扩展正则需要加上选项 -r。

**1、符号"?"**
  
`?`：表示前置字符有 0 个或 1 个。

**2、符号"+"**

`+`：表示前置字符有 1 个或多个。


**3、符号|**
* `|`：表示指明两项之间的一个选择。
* `abc|ABC`：表示可以匹配 abc 或者 ABC。

**4、符号"()"**

* `()` 表示分组，类似算数表达式中的 ()。子命令表达式中可以通过 \1，\2，\3 等来表示分组匹配到的内容。其实 "()" 也可以在基本正则表达式中使用的。
* `(a|b)b`：表示可以匹配 ab 或者 bb 字串
* `([0-9])|([0][0-9])|([1][0-9])`：表示匹配 0-9 或者 00-09 或者 10-19 范围的字符。

**5、符号"{}"**

这里的 `{}` 和基本正则表达式中的大括号意义是一样的，只不过在使用时不用加 `\` 转义符号。
