---
layout: post
title:  "sed入门教程系列之：sed语法和常用选项"
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
* **sed 入门教程系列 04：sed语法和常用选项**
* [sed 入门教程系列 05：数字定址和正则定址](/articles/sed-tutorial-series-05.html)
* [sed 入门教程系列 06：基本子命令](/articles/sed-tutorial-series-06.html)
* [sed 入门教程系列 07：sed工作模式](/articles/sed-tutorial-series-07.html)
* [sed 入门教程系列 08：高级子命令](/articles/sed-tutorial-series-08.html)
* [sed 入门教程系列 09：分支和测试](/articles/sed-tutorial-series-09.html)
* [sed 入门教程系列 10：sed实战练习](/articles/sed-tutorial-series-10.html)

## sed语法和常用选项

**1、语法**
```
sed [选项] 'command' 文件名称
选项部分，常见选项包括 -n，-e，-i，-f，-r 选项。
command部分包括：[地址1，地址2] [函数] [参数(标记)]
```

**2、常用选项-n**

sed 默认会把模式空间处理完毕后的内容输出到标准输出，也就是输出到屏幕上，加上 -n 选项后被设定为安静模式，也就是不会输出默认打印信息，除非子命令中特别指定打印选项，则只会把匹配修改的行进行打印。

例子1：
```bash
echo -e 'hello world\nnihao' | sed 's/hello/A/'
# 结果：
A world
nihao
```

例子2 ：
 ```bash
echo -e 'hello world\nnihao' | sed -n 's/hello/A/'
# 结果：加 -n 选项后什么也没有显示。
```

例子3：
```bash
echo -e 'hello world\nnihao' | sed -n 's/hello/A/p'
# 结果：A world/
```
说明：-n 选项后，再加 p 标记，只会把匹配并修改的内容打印了出来。


**3、常用选项-e**

如果需要用 sed 对文本内容进行多种操作，则需要执行多条子命令来进行操作。

例子1：
```bash
echo -e 'hello world' | sed -e 's/hello/A/' -e 's/world/B/'
# 结果：A B
```

例子2：
```bash
echo -e 'hello world' | sed 's/hello/A/;s/world/B/'
# 结果：A B
```
说明：例子 1 和例子 2 的写法的作用完全等同，可以根据喜好来选择，如果需要的子命令操作比较多的时候，无论是选择 -e 选项方式，还是选择分号的方式，都会使命令显得臃肿不堪，此时使用 -f 选项来指定脚本文件来执行各种操作会比较清晰明了。

**4、常用选项-i**

sed 默认会把输入行读取到模式空间，简单理解就是一个内存缓冲区，sed 子命令处理的内容是模式空间中的内容，而非直接处理文件内容。因此在 sed 修改模式空间内容之后，并非直接写入修改输入文件，而是打印输出到标准输出。如果需要修改输入文件，那么就可以指定 -i 选项。

例子1：
```bash
cat file.txt
hello world
$ sed 's/hello/A/' file.txt
A world
$ cat file.txt
hello world
```
例子2：
```bash
$ sed -i 's/hello/A/' file.txt
$ cat file.txt
A world
```

例子3：
```bash
$ sed –i.bak 's/hello/A/' file.txt
```
说明：最后一个例子会把修改内容保存到 file.txt，同时会以 file.txt.bak 文件备份原来未修改文件内容，以确保原始文件内容安全性，防止错误操作而无法恢复原来内容。

**5、常用选项-f**

还记得 -e 选项可以来执行多个子命令操作，用分号分隔多个命令操作也是可以的，如果命令操作比较多的时候就会比较麻烦，这时候把多个子命令操作写入脚本文件，然后使用 -f 选项来指定该脚本。

例子1：
```bash
echo "hello world" | sed -f sed.script
# 结果：A B
# sed.script 脚本内容：
s/hello/A/
s/world/B/
```
说明：在脚本文件中的子命令串就不需要输入单引号了。

**6、常用选项-r**

sed 命令的匹配模式支持正则表达式的，默认只能支持基本正则表达式，如果需要支持扩展正则表达式，那么需要添加 -r 选项。

例子1：
```bash
echo "hello world" | sed -r 's/(hello)|(world)/A/g'
A A
```
