---
layout: post
title:  "软件开发之code review"
categories: 软件开发
tags: 软件开发 CodeReview
author: 肖邦
---

* content
{:toc}

> 在之前没有怎么正式做过代码的 code review，我相信国内有很多公司也是不做 code review 的。其实在国外 Review 是软件项目开发的非常重要流程之一，它关乎到系统代码的质量、健壮性、可维护性等等，异常重要。文中总结了一下关于 Review 的一些知识点，以后根据实际经验再更新。




## code_review 作用
* **通过大家的建议增进代码的质量**。如，程序的可读性、可维护性、设计与实现、逻辑与思路。
* code_review 是一个**传递知识的手段**，可以让其他并不熟悉代码的人知道作者的意图和想法，从而可以在以后轻松维护代码。
* 鼓励程序员们相互**学习对方的长处和优点**。


## code_review list
* 可读性
* 可维护性
  - 接口设计是否合理
  - 变量不能为魔鬼数字
  - 模块结构是否合理
  - 外部软件的依赖
* 资源泄露
  - 内存泄露
  - 忘关闭文件描述符
  - 未及时关闭socket连接
* 程序的逻辑
* 对需求和设计的实现
* review 后评价代码是否会产生技术负债？也就是可能会给未来埋下隐患之处。比如：程序中使用了一个较新(很不常用)的系统或语言特性，而在文档中没有详细说明，容易给后续维护人员挖坑。
* 代码实现的优雅程度。这个是一种感觉，比方说：实现逻辑饶不饶，是否足够轻巧。
* 代码风格和编码标准
  - 变量命名
  - 缩进方式
  - 大括号是否另起一行
* 发现代码的 Bug


## code_review 不应该
* **不应该承担发现代码错误的职责**。code_review 主要是审核代码的质量，如可读性，可维护性，以及程序的逻辑和对需求和设计的实现，而代码的 Bug 应该由单元测试和功能测试来保证。
* **不应该成为保证代码风格和编码标准的手段**。每个程序员在把自己的代码提交团队 Review 的时候，代码就应该是符合规范的，这是默认值。
* 并不是说，你不能在 Code Review 中报告一个程序的 bug 或是一个代码规范的问题，而是说那并不是 Code Review 的意图。


## 参考链接
* 酷壳 CoolShell 的 《[从CODE REVIEW 谈如何做技术](https://coolshell.cn/articles/11432.html)》
* 酷壳 CoolShell 的 《[CODE REVIEW 中的几个提示](https://coolshell.cn/articles/1302.html)》
* 酷壳 CoolShell 的 《[简单实用的 CODE REVIEW 工具](https://coolshell.cn/articles/1218.html)》
* 朱赟的 《[说说 Code Review](https://mp.weixin.qq.com/s?__biz=MzA4ODgwNjk1MQ==&mid=2653788392&idx=1&sn=e84f8546dff32030cef89fc29680db80&mpshare=1&scene=1&srcid=1029dWTXWxiFjLaC7NORW97l#rd)》
