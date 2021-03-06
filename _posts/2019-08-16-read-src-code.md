---
layout: post
date: "2019-08-16 22:13:15"
title:  "如何高效学习和阅读代码"
categories: 软件开发
tags: code
author: 肖邦
---

* content
{:toc}

> 很多时候我们会接手别人写的项目代码，或者是为了学习某些优秀的开源项目，我们需要阅读源代码，一般项目代码都比较复杂，总结出一种有用、高效的方法套路非常重要，能够很大程度提高我们的工作或学习效率。




## 读文档还是读代码

杰夫·阿特伍德说过一句话：[Code Tells You How, Comments Tell You Why](https://blog.codinghorror.com/code-tells-you-how-comments-tell-you-why/)。扩展一下：
* `代码`：What + How + Detail
* `文档/书`： What + How + Why

代码并不会告诉你`Why`，看代码只能靠猜测或推导来估计`Why`，是揣测，不准确，所以会有很多误解。而且，我们知道 `Why` 是能让人一通百通、醍醐灌醒的东西。但代码会告诉你细节，这是书和文档不能给你的，也有细节是魔鬼，细节决定成败的说法。书和文档是人对人说的话，代码是人对机器说的话，所以：

* 如果你想知道人为什么要这么搞，那么应该去看书，看文档。
* 如果你想知道机器干了什么，那么应该去看代码。
* 如果你想了解一种思想，一种方法，一种原理，一种思路，一种经验，读书和读文档会更高效些
* 如果你想了解的是具体的细节，比如某协程的实现，某个模块的性能，某个算法的实现，那么还是要去读代码的。因为代码中会有更具体的细节处理，尤其是一些 `edge case` 或`代码技巧`方面的内容。

从代码中收获大，还是从书中的收获大，在不同的场景、不同的目的下，会有不同的答案。
* 如果你是个新手，那应该多读代码，多动手写代码，你需要的是感性认识。
* 如果你是个老手，你的成长需要更多的 `理性认识`，这个阶段你会喜欢读好书和文章。




## 如何阅读源代码

在阅读源代码之前，最好有下面的前提再去阅读代码，这样读起来会很顺畅。

* **基础知识**。相关的语言和基础技术的知识。
* **软件功能**。软件完成的功能，有哪些特性，哪些配置项。最好要读一遍用户手册，让软件跑起来，自己使用感受一下。
* **相关文档**。读一下相关的内部文档，`Readme` ，`Release Note`，`Design`，`Wiki`，这些文档可以让你整明白软件的方方面面。
* **代码的组织结构**。代码目录中每个目录是什么样的功能，每个文档是干什么的，如果有的话最好看下 `Makefile`。


接下来要了解，软件的代码是有哪些部分构成的，这里给出个列表，供参考：

* **接口抽象定义**。其描述了代码需要处理的数据结构或业务实体，以及它们之间的关系。
* **模块粘合层**。我们代码有很多是用来粘合代码的，比如中间件、回调、代理委托、依赖注入等。
* **业务流程**。这是代码运行的过程，在这个流程中，数据怎么被传递和处理的，一般我们需要画程序流程图和时序处理图。
* **具体实现**。对代码的实现，一般需要下面一些事实。
  * **代码逻辑**。代码有两种逻辑，一种是业务逻辑，真正处理业务逻辑，另一种是控制逻辑，只是用于控制程序流转的，不是业务逻辑。比如：flag 控制变量、多线程处理的代码、异步控制的代码、通信的代码等，这两种逻辑你要分开。
  * **出错处理**。通常按照二八原则，20% 代码是正常的逻辑，80% 代码是在处理各种错误。所以在读代码时，可以把代码处理错误的部分全部删除掉。可以排除干扰因素，更高效阅读代码。
  * **数据处理**。好多代码都是在处理数据流，这些代码冗长无聊，不是主要逻辑，可以先不理。
  * **重要的算法**。核心的算法可能会比较难读，往往是最具有技术含量的部分。
  * **底层交互**。读这些代码通常需要一定的底层技术知识，不然很难读懂。
* **运行时调试**。让代码运行起来，通过日志或 debug 设置断点跟踪也好，实际看下代码运行过程，是了解代码的很好方式。



总结一下阅读代码的方法：

* 一般采用自顶向下，从总体到细节的 `剥洋葱皮` 的读法。
* 画图是必要的，程序流程图、调用时序图、模块组织图。
* 代码逻辑归类，排除杂音，主要逻辑才会更清楚。
* debug 跟踪一下代码是了解代码在执行中发生了什么的最好方式。



## 如何高效学习开源项目

对于技术人员来说，开源项目如果只是「拿来主义」，那么对技术提升没有什么本质上的帮助。对于开源项目，要深入地去学习，要做到，知其然，知其所以然。一方面是为了更好的应用这些开源项目，另一方面是为了通过学习优秀的开源项目来提升自己的能力。下边结合自己的经验谈谈对如何学习开源项目的一些看法：

* 需要树立正确的观念：不管你是什么身份，都可以从开源项目中学习到很多东西。
* 不要 `只` 盯着数据结构与算法。
* 采用 `自顶向下`的学习方法，**源码不是第一步，而是最后一步**。不要一上来就看源码，而是要基本掌握了功能、原理、关键设计之后再去看源码。看源码的主要目的是为了学习代码的写作方式，以及关键技术的实现。



下边详细总结下 `自顶向下` 的学习方法和步骤：

* **第一步：安装**
    * 可以获取系统的依赖组件，依赖的组件是系统设计和实现的基础。
    * 安装目录也可以能够提供一些使用和运行的基本信息。
    * 系统提供了哪些工具方便我们使用。
    * 带着问题学习效率是最高的。
* **第二步：运行**
    * 关注命令行
    * 关注配置文件
    * 它们提供了两个非常关键的信息：系统具备哪些能力和系统将会如何运行。
    * 尝试修改配置，看系统有什么变化。
    * 查看日志，Debug 信息等方式。
* **第三步：原理研究**
    * 关键特性的基本实现原理
    * 优缺点对比分析。只有清楚掌握技术方案的优缺点后才算真正的掌握这门技术，才能在架构设计时作出合理的选择。
    * 通读项目的设计文档
    * 阅读网上已有的分析文档
    * Demo 验证。如果一些技术点难查到资料，可以自己写一些 Demo 验证，通过日志或 Debug 信息，这样能够清晰理解具体的细节。
* **第四步：测试**
    * **如果应用到线上的开源项目，一定要自己执行测试**。如果是为了学习开源项目，可以从网上找一些分析和测试文档。但如果是应用到生产环境的开源项目，一定要自己进行测试，因为网上的测试结果不一定契合自己的业务场景。不同的软件版本、系统环境、测试方法、测试用例，都可能得出不同的结论。不过构建完整的测试用例，既耗费大量时间，又需要较多机器资源，成本投入会变得比较大。
    * **测试一定要在原理理解之后，不能安装完成之后立马测试**。原因在于如果对系统不熟悉，很可能出现命令行、并非最佳配置参数，导致没有根据业务特点搭建正确的环境、没有设计合理的测试用例，从而通过最终的测试结果得出错误的结论，误导了错误的决策。
* **第五步：源码研究**
    * 源码研究的主要目的是学习原理背后的具体编码如何实现，通过学习这些技巧来提升我们自己的技术能力。
    * 尽量不要尝试通读所有的代码。带着明确的目的去研究源码，做到有的放矢，才能事半功倍。
    * 写 Demo 程序调用基础库完成一些简单的功能，然后通过调试来看具体的调用栈，通过调用栈来理解基础库的处理逻辑和过程，这样会比单纯看代码更高效。

