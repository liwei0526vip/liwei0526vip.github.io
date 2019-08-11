---
layout: post
date: "2019-08-11 9:36:42"
title:  "性能分析工具之火焰图"
categories: Linux工具
tags: 火焰图 perf 性能分析
author: 肖邦
---

* content
{:toc}

> 记录 Linux 系统下性能分析时相关工具火焰图使用方法。常见性能分析工具：perf、systemtap、火焰图。。。本文主要总结火焰图的基本使用方法。




## 火焰图说明

[Brendan D. Gregg](http://www.brendangregg.com/perf.html#FlameGraphs) 发明了 [`火焰图`](http://www.brendangregg.com/flamegraphs.html)。

常见的火焰图类型有：
* [On-CPU](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html)
* [Off-CPU](http://www.brendangregg.com/FlameGraphs/offcpuflamegraphs.html)
* [Memory](http://www.brendangregg.com/FlameGraphs/memoryflamegraphs.html)
* [Hot / Cold](http://www.brendangregg.com/FlameGraphs/hotcoldflamegraphs.html)
* [Differential](http://www.brendangregg.com/blog/2014-11-09/differential-flame-graphs.html)

生成火焰图大概过程如下：
* 采集系统信息
* 处理信息
* 生成火焰图


## 生成火焰图

以最近研究的项目 dpvs 为例，记录如何 `On-CPU` 类型的火焰图。

* 可视化生成器
  ```sh
  $ git clone https://github.com/brendangregg/FlameGraph.git
  ```
* 采集数据（perf）
  ```sh
  $ perf record -p `pidof dpvs` -g -- sleep 30
  ```
* 生成火焰图
  ```sh
  # 生成折叠后的调用栈
  $ perf script -i perf.data &> perf.unfold

  # 将 perf 解析出的内容 perf.unfold 中的符号进行折叠
  $ stackcollapse-perf.pl perf.unfold &> perf.folded

  # 将 perf 解析出的内容 perf.unfold 中的符号进行折叠
  $ flamegraph.pl perf.folded > perf.svg

  # 我们可以使用管道将上面的流程简化为一条命令
  $ perf script | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > process.svg
  ```


## 解析火焰图

生成的火焰图是 svg 格式，如下图所示：

![火焰图](/image/perf.svg)

**一、火焰图的含义**

火焰图是基于 stack 信息生成的 SVG 图片, 用来展示 CPU 的调用栈。
* y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。
* x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长。注意，x 轴不代表时间，而是所有的调用栈合并后，按字母顺序排列的。
* 火焰图就是看顶层的哪个函数占据的宽度最大。只要有 “平顶”(plateaus)，就表示该函数可能存在性能问题。
* 颜色没有特殊含义，因为火焰图表示的是 CPU 的繁忙程度，所以一般选择暖色调。


**二、互动性**

火焰图是 SVG 图片，可以与用户互动。
* **鼠标悬浮**。火焰的每一层都会标注函数名, 鼠标悬浮时会显示完整的函数名、抽样抽中的次数、占据总抽样次数的百分比
* **点击放大**。在某一层点击，火焰图会水平放大，该层会占据所有宽度，显示详细信息。左上角会同时显示 “Reset Zoom”，点击该链接，图片就会恢复原样。
* **搜索**。按下 Ctrl + F 会显示一个搜索框，用户可以输入关键词或正则表达式，所有符合条件的函数名会高亮显示。

**三、局限**

两种情况下，无法画出火焰图，需要修正系统行为。
* **调用栈不完整**。当调用栈过深时，某些系统只返回前面的一部分（比如前10层）。
* **函数名缺失**。有些函数没有名字，编译器只用内存地址来表示（比如匿名函数）。


参考：
> * [https://blog.csdn.net/gatieme/article/details/78885908](https://blog.csdn.net/gatieme/article/details/78885908)
> * [https://blog.csdn.net/juS3Ve/article/details/85110359](https://blog.csdn.net/juS3Ve/article/details/85110359)
> * [https://moonbingbing.gitbooks.io/openresty-best-practices/content/flame_graph.html](https://moonbingbing.gitbooks.io/openresty-best-practices/content/flame_graph.html)
