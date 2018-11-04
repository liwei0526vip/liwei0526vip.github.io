---
layout: post
title:  "sed入门教程系列之：基本子命令"
categories: Linux工具
tags: sed
author: 肖邦
---

* content
{:toc}

> sed 是一个比较古老的，功能十分强大的用于文本处理的流编辑器，加上正则表达式的支持，可以进行大量的复杂的文本编辑操作。sed 本身是一个非常复杂的工具，有专门的书籍讲解 sed 的具体用法，但是个人觉得没有必要去学习它的每个细节，那样没有特别大的实际意义。网上也有很多关于 sed 的教程，我也是以学以致用的心态来学习 sed 的常见的用法，并进行系统的总结，内容基本覆盖了 sed 的大部分的知识点。文中的内容比较简练，加以实际示例来帮助去理解 sed 的使用。




本系列文章目录：
* [sed 入门教程系列 01：基本原理介绍](/articles/sed-tutorial-series-00.html)
* [sed 入门教程系列 02：基本正则表达式](/articles/sed-tutorial-series-02.html)
* [sed 入门教程系列 03：扩展正则表达式](/articles/sed-tutorial-series-03html)
* [sed 入门教程系列 04：sed语法和常用选项](/articles/sed-tutorial-series-04.html)
* [sed 入门教程系列 05：数字定址和正则定址](/articles/sed-tutorial-series-05.html)
* **sed 入门教程系列 06：基本子命令**
* [sed 入门教程系列 07：sed工作模式](/articles/sed-tutorial-series-07.html)
* [sed 入门教程系列 08：高级子命令](/articles/sed-tutorial-series-08.html)
* [sed 入门教程系列 09：分支和测试](/articles/sed-tutorial-series-09.html)
* [sed 入门教程系列 10：sed实战练习](/articles/sed-tutorial-series-10.html)

## 七、基本子命令

**1、子命令a**

子命令 a 表示在指定行下边插入指定行的内容。

例子1：
```bash
$ sed 'a A' message
```
说明：将 message 文件中每一行下边都插入添加一行内容是 A。

例子2：
```bash
$ sed '1,2a A' message
```
说明：将 message 文件中 1-2 行的下边插入添加一行内容是 A

例子3：
```bash
$ sed '1,2a A\nB\nC' message
```
说明：将 message 文件中 1-2 行的下边分别添加 3 行，3 行内容分别是A、B、C，这里使用了 \n，插入多行内容都可以按照这种方式来实现。

**2、子命令i**

子命令 i 和 a 使用上基本上一样，只不过是在指定行上边插入指定行的内容。

例子1：
```bash
$ sed 'i A' message
```
说明：将 message 文件中每一行上边都插入添加一行内容是 A。

例子2：
```bash
$ sed '1,2i A' message
```
说明：将 message 文件中 1-2 行的上边插入添加一行内容是 A

例子3：
```bash
$ sed '1,2i A\nB\nC' message
```
说明：将 message 文件中 1-2 行的上边分别添加 3 行，3 行内容分别是 A、B、C，这里使用了 \n，插入多行内容都可以按照这种方式来实现。

**3、子命令c**

子命令 c 是表示把指定的行内容替换为自己需要的行内容。

例子1：
```bash
$sed 'c A' message
```
说明：将 message 文件中所有的行内容都分别替换为 A 行内容。

例子2：
```bash
$ sed '1,2c A' message
```
说明：将 message 文件中 1-2 行的内容替换为 A，注意这里说的是将 1-2 行所有的内容只替换为一个 A 内容，也就是 1-2 行内容编程了一行，定址如果连续就是这种情况。如果想把 1-2 行分别替换为 A，可以参考下个例子的方式。

例子3：
```bash
$ sed '1,2c A\nA' message
```
说明：将 message 中 1-2 行内容分别替换为了 A，需要在替换内容上手动加换行 \n，这样当然也可以将一行内容替换为多行内容。

**4、子命令d**

子命令 d 表示删除指定的行内容，比较简单，更容易理解。

例子1：
```bash
$ sed 'd' message
```
说明：将 message 所有行全部删除，因为没有加定址表达式，所以平时如果需要删除指定行内容，需要在子命令前加定址表达式。

例子2：
```
$ sed '1,3d' message
```
说明：将 message 文件中 1-3 行内容删除。

**5、子命令y**

子命令 y 表示字符替换，可以替换多个字符，只能替换字符不能替换字符串，且不支持正则表达式，具体使用方法看例子。

例子1：
```bash
$ sed 'y/ab/AB/' message
```
说明：把 message 中所有 a 字符替换为 A 符号，所有 b 字符替换为 B 符号。强调一下，这里的替换源字符个数和目的字符个数必须相等；字符不支持正则表达式；源字符和目标字符每个字符需要一一对应。

**6、子命令=**

子命令 =，可以将行号打印出来。

例子1：
```bash
$ sed '1,2=' message
# 结果：
1
nihao
2
hello world
```
说明：将指定行的上边显示行号。

**7、子命令r**

子命令 r，类似于 a，也是将内容追加到指定行的后边，只不过 r 是将指定文件内容读取并追加到指定行下边。 

例子1：
```
$ sed '2r a.txt' message
```
说明：将 a.txt 文件内容读取并插入到 message 文件第 2 行的下边。

**8、子命令s**

子命令 s 为替换子命令，是平时 sed 使用的最多的子命令，没有之一。因为支持正则表达式，功能变得强大无比，下边来详细地说说子命令 s 的使用方法。

* 基本语法：
  ```
  [address]s/pattern/replacement/flags
  ```
* s 字符串替换，替换的时候可以把 / 换成其它的符号，比如 =，replacement 部分用下列字符会有特殊含义：
  ```
  &：用正则表达式匹配的内容进行替换
  \n：回调参数
  \(\)：保存被匹配的字符以备反向引用 \n 时使用，最多 9 个标签，标签书序从左到右
  ```
* Flags
    ```
    n：可以是 1-512，表示第 n 次出现的情况进行替换
    g：全局更改
    p：打印模式空间的内容
    w file：写入到一个文件 file 中
    ```

测试文件：
```bash
$ cat message
hello 123 world
```
例子1：
```bash
$ sed 's/hello/HELLO/' message
```
说明：将 message 每行包含的第一个 hello 的字符串替换为 HELLO，这是最基本的用法。

例子2：
```bash
$ sed -r 's/[a-z]+ [0-9]+ [a-z]+/A/' message
# 结果：A
```
说明：使用了扩展正则表达式，需要加 -r 选项。

例子3：
```bash
$ sed -r 's/([a-z]+)( [0-9]+ )([a-z]+)/\1\2\3/' message
# 结果：hello 123 world
```
说明：再看下一个例子就明白了。

例子4：
```bash
$ sed -r 's/([a-z]+)( [0-9]+ )([a-z]+)/\3\2\1/' message
# 结果：world 123 hello
```
说明：\1 表示正则第一个分组结果，\2 表示正则匹配第二个分组结果，\3 表示正则匹配第三个分组结果。

例子5：
```bash
$ sed -r 's/([a-z]+)( [0-9]+ )([a-z]+)/&/' message
# 结果：hello 123 world
```
说明：& 表示正则表达式匹配的整个结果集。

例子6：
```bash
$ sed -r 's/([a-z]+)( [0-9]+ )([a-z]+)/111&222/' message
# 结果：111hello 123 world222
```
说明：在匹配结果前后分别加了 111、222。

例子7：
```
$ sed -r 's/.*/111&222/' message
```
说明：在 message 文件中每行的首尾分别加上 111、222。

例子8：
```bash
$ sed 's/i/A/g' message
```
说明：把 message 文件中每行的所有 i 字符替换为 A，默认不加 g 标记时只替换每行的第一个字符。

例子9：
```bash
$ sed 's/i/A/2' message
```
说明：把 message 文件中每行的第 2 个 i 字符替换为 A。

例子10：
```bash
$ sed -n 's/i/A/p' message
```
说明：加 -p 标记会把被替换的行打印出来，再加上 -n 选项会关闭模式空间打印模式，因此该命令的效果就是只显示被替换修改的行。

例子11：
```bash
$ sed -n 's/i/A/w b.txt' message
```
说明：把 message 文件中内容的每行第一个字符 i 替换为 A，然后把修改内容另存为 b.txt 文件。

例子12：
```bash
$ sed -n 's/i/A/i' message
```
说明：把 message 文件中每一行的第一个 i 或 I 字符替换为 A 字符，也即是忽略大小写。
 
