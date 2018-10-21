---
layout: post
title:  "C 语言标准库 IO 操作详解"
categories: C语言开发
tags:  IO 
author: 肖邦
---

* content
{:toc}

> 其实输入与输出对于不管什么系统的设计都是异常重要的，比如设计 C 接口函数，首先要设计好输入参数、输出参数和返回值，接下来才能开始设计具体的实现过程。C 语言标准库提供的接口功能很有限，不像 Python 库。不过想把它用好也不容易，本文总结 C 标准库基础 IO 的常见操作和一些特别需要注意的问题，如果你觉着自己还不是大神，那么请相信我，读完全文后你肯定会有不少收获。




## 一、操作句柄
打开文件其实就是在操作系统中分配一些资源用于保存该文件的状态信息及文件的标识，以后用户程序可以用这个标识做各种读写操作，关闭文件则释放占用的资源。

打开文件的函数：
```cpp
#include <stdio.h>
FILE *fopen(const char *path, const char *mode);
```
FILE 是 C 标准库定义的结构体类型，其包含文件在内核中的标识（文件描述符）、I／O 缓冲区和当前读写位置信息，调用者不需知道 FILE 的具体成员，由库函数内部维护，调用者不应该直接访问这些成员。像 FILE* 这样的文件指针称为句柄（Handle）。

打开文件操作是对文件资源进行操作的，所以有可能打开文件失败，所以在打开函数时一定要判断返回值，如果失败则返回错误信息，以方便快速定位错误。

打开文件应该与关闭文件成对存在，虽然程序在退出时会释放相应的资源，但是对于一个长时间运行服务程序来说，经常打开而不关闭文件是会造成进程资源耗尽的，因为进程的文件描述符个数是有限的，及时关闭文件是个好习惯。

关闭文件的函数：
```cpp
#include <stdio.h>
int fclose(FILE *fp);
```

fopen 函数参数 mode 总结：
* "r"：只读，文件必须存在。
* "w"：只写，如果不存在则创建，存在则覆盖。
* "a"：追加，如果不存在则创建。
* "r+"：允许读和写，文件必须存在。
* "w+"：允许读和写，文件不存在则创建，存在则覆盖。
* "a+"：允许读和追加，文件不存在则创建。


## 二、关于stdin/stdout/stderr
在用户程序启动时，main 函数还没开始执行之前，会自动打开三个 FILE* 指针分别是：stdin、stdout、stderr，这三个文件指针是 libc 中定义的全局变量，在 stdio.h 中声明，printf 向 stdout 写，而 scanf 从 stdin 读，用户程序也可以直接使用这三个文件指针。

- stdin 只用于读操作，称为标准输入
- stdout 只用于写操作，称为标准输出
- stderr 也用于写操作，称为标准错误输出

通常程序的运行结果打印到标准输出，而错误提示打印到标准错误输出，一般标准输出和标准错误都是屏幕。通常可以标准输出重定向到一个常规文件，而标准错误输出仍然对应终端设备，这样就可以将运行结果与错误信息分开。


## 三、以字节为单位的IO函数
fgetc 函数从指定的文件中读一个字节，getchar从标准输入读一个字节，调用 getchar() 相当于 fgetc(stdin)
```cpp
#include <stdio.h>
int fgetc(FILE *stream);
int getchar(void);
```

fputc 函数向指定的文件写入一个字节，putchar 向标准输出写一个字节，调用 putchar() 相当于调用 fputc(c, stdout)。
```cpp
#include <stdio.h>
int fputc(int c, FILE *stream);
int putchar(int c);
```

参数和返回值类型为什么使用 int 类型？可以看到这几个函数的参数和返回值类型都是 int，而非 unsigned char 型。因为错误或读到文件末尾时将返回 EOF，即 -1，如果返回值是 unsigned char（0xff），与实际读到字节 0xff 无法区分，如果使用 int 就可以避免这个问题。

## 四、操作读写位置函数
当我们在操作文件时，有一个叫「文件指针」的家伙来记录当前操作的文件位置，比如刚打开文件，调用了 1 次 fgetc 后，此时文件指针指向了第 1 个字节后边，注意是以字节为单位记录的。

改变文件指针位置的函数：
```cpp
#include <stdio.h>
int fseek(FILE *stream, long offset, int whence);
// whence：从何处开始移动，取值：SEEK_SET | SEEK_CUR | SEEK_END
// offset：移动偏移量，取值：可取正 | 负
void rewind(FILE *stream);
```
举几个简单例子：
```cpp
fseek(fp, 5, SEEK_SET);     // 从文件头向后移动5个字节
fseek(fp, 6, SEEK_CUR);     // 从当前位置向后移动6个字节
fseek(fp, -3, SEEK_END);    // 从文件尾向前移动3个字节
```

offset 可正可负，负值表示向文件开头的方向移动，正值表示向文件尾方向移动，如果向前移动的字节数超过文件开头则出错返回，如果向后移动的字节数超过了文件末尾，再次写入会增加文件尺寸，文件空洞字节都是 0

写入测试数据：
```sh
$ echo "5678" > file.txt
```
设置偏移量并写入文件内容：
```cpp
fp = fopen("file.txt", "r+");
fseek(fp, 10, SEEK_SET);
fputc('K', fp)
fclose(fp)
```
通过结果可以看出字母K是从第10个位置开始写的
```sh
liwei:/tmp$ od -tx1 -tc -Ax file.txt 
0000000    35  36  37  38  0a  00  00  00  00  00  4b                    
           5   6   7   8  \n  \0  \0  \0  \0  \0   K
```

`rewind(fp)` 等价于 `fseek(fp, 0, SEEK_SET)`

`ftell(fp)` 函数比较简单，直接返回当前文件指针在文件中的位置
```cpp
// 实现计算文件字节数的功能
fseek(fp, 0, SEEK_END);
ftell(fp);
```

## 五、以字符串为单位的IO函数
fgets 从指定的文件中读一行字符到调用者提供的缓冲区，读入内容不超过 size 。
```cpp
char *fgets(char *s, int size, FILE *stream);
char *gets(char *s);
```

首先要说明 gets() 函数强烈不推荐使用，类似 strcpy 函数，用户不可以指定缓冲区大小，很容易造成缓冲区溢出错误。不过 strcpy 程序员还是可以避免，而 gets 的输入用户可以提供任意长的字符串，唯一避免方法就是不使用 gets，而使用 fgets(buf, size, stdin)

fgets 函数从 stream 所指文件读取以 '\n' 结尾的一行，包括 '\n' 在内，存到缓冲区中，并在该行结尾添加一个 '\0' 组成完整的字符串。如果文件一行太长，fgets 从文件中读了 size-1 个字符还没有读到 '\n'，就把已经读到的 size-1 个字符和一个 '\0' 字符存入缓冲区，文件行剩余的内容可以在下次调用 fgets 时继续读。

若一次 fgets 调用在读入若干字符后到达文件末尾，则将已读到的字符加上 '\0' 存入缓冲区并返回，如果再次调用则返回 NULL，可以据此判断是否读到文件末尾。

fputs 向指定文件写入一个字符串，缓冲区保存的是以 '\0' 结尾的字符串，与 fgets 不同的是，fputs 不关心字符串中的 '\n' 字符。
```cpp
int fputs(const char *s, FILE *stream);
int puts(const char *s);
```

## 六、以记录为单位的IO函数
```cpp
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```
fread 和 fwrite 用于读写记录，这里的记录是指一串固定长度的字节，比如一个 int、一个结构体货或一个定长数组。

参数 size 指出一条记录的长度，nmemb 指出要读或写多少条记录，这些记录在 ptr 所指内存空间连续存放，共占 size * nmemb 个字节。

fread 和 fwrite 返回的记录数有可能小于 nmemb 指定的记录数。例如当读写位置距文件末尾只有一条记录长度，调用 fread 指定 nmemb 为 2，则返回值为 1。如果写文件时出错，则 fwrite 的返回值小于 nmemb 指定的值。
```cpp
struct t{
    int   a;
    short b;
};
struct t val = {1, 2};
FILE *fp = fopen("file.txt", "w");
fwrite(&val, sizeof(val), 1, fp);
fclose(fp);
```
```sh
liwei:/tmp$ od -tx1 -tc -Ax file.txt 
0000000    01  00  00  00  02  00  00  00                                
         001  \0  \0  \0 002  \0  \0  \0
```
从结果可以看出，写入的是 8 个字节，有兴趣的同学可以就此分析下系统的「大小端」和结构体的「对齐补齐」问题。

## 七、格式化IO函数
### printf/scanf
```cpp
int printf(const char *format, ...);
int scanf(const char *format, ...);
```
这两个函数是我们学习 C 语言最早接触，可能也是接触比较多的了，没什么特别要说的。printf 就是格式化打印到标准输出。下面总结下 printf 常用的方式。
```cpp
printf("%d\n", 5);            // 打印整数 5
printf("-%10s-\n", "hello")   // 设置显示宽度并左对齐：-     hello-
printf("-%-10s-\n", "hello")  // 设置显示宽度并右对齐：-     hello-
printf("%#x\n", 0xff);        // 0xff 不加#则显示ff
printf("%p\n", main);         // 打印 main 函数首地址
printf("%%\n");               // 打印一个 %
```

scanf 就是从标准输入中读取格式化数据，简单举个例子：
```cpp
int year, month, day;
scanf("%d/%d/%d", &year, &month, &day);
printf("year = %d, month = %d, day = %d\n", year, month, day);
```

### sprintf/sscanf/snprintf
sprintf 并不打印到文件，而是打印到用户提供的缓冲区中并在末尾加 '\0'，由于格式化后的字符串长度很难预计，所以很可能造成缓冲区溢出，强烈推荐 snprintf 更好一些，参数 size 指定了缓冲区长度，如果格式化后的字符串超过缓冲区长度，snprintf 就把字符串截断到 size - 1 字节，再加上一个 '\0'，保证字符串以 '\0' 结尾。如果发生截断，返回值是截断之前的长度，通过对比返回值与缓冲区实际长度对比就知道是否发生截断。
```cpp
int sscanf(const char *str, const char *format, ...);
int sprintf(char *str, const char *format, ...);
int snprintf(char *str, size_t size, const char *format, ...);
```


sscanf 是从输入字符串中按照指定的格式去读取相应的数据，函数功能非常的强大，支持类似正则表达式匹配的功能。具体的使用格式请自行查询官方手册，这里总结出最常用、最重要的几种使用场景和方式。
* 最基本的用法
```cpp
char buf[1024] = 0;
sscanf("123456", "%s", buf);
printf("%s\n", buf);
// 结果为：123456
```
* 取指定长度的字符串
```cpp
sscanf("123456", "%4s", buf);
printf("%s\n", buf);
// 结果为：1234
```
* 取第1个字符串
```cpp
sscanf("hello world", "%s", buf);
printf("%s\n", buf);
// 结果为：hello  因为默认是以空格来分割字符串的，%s读取第一个字符串hello
```
* 读取到指定字符为止的字符串
```cpp
sscanf("123456#abcdef", "%[^#]", buf);
// 结果为：123456
// %[^#]表示读取到#符号停止，不包括#
```
* 读取仅包含指定字符集的字符串
```cpp
sscanf("123456abcdefBCDEF", "%[1-9a-z]", buf);
// 结果为：123456abcdef
// 表达式是要匹配数字和小写字母，匹配到大写字母就停止匹配了。
```
* 读取指定字符集为止的字符串
```cpp
sscanf("123456abcdefBCDEF", "%[^A-Z]", buf);
// 结果为：123456abcdef
```
* 读取两个符号之间的内容(@和.之间的内容)
```cpp
sscanf("liwei0526vip@linuxblogs.cn", "%*[^@]@%[^.]", buf);
// 结果为：linuxblogs
// 先读取@符号前边内容并丢弃，然后读@，接着读取.符号之前的内容linuxblogs，不包含字符.
```
* 给一个字符串
```cpp
sscanf("hello, world", "%*s%s", buf);
// 结果为：world
// 先忽略一个字符串"hello,"，遇到空格直接跳过，匹配%s，保存 world 到 buf
// %*s 表示第 1 个匹配到的被过滤掉，即跳过"hello,"，如果没有空格，则结果为 NULL
```
* 稍微复杂点的
```cpp
sscanf("ABCabcAB=", "%*[A-Z]%*[a-z]%[^a-z=]", buf);
// 结果为：AB  自己尝试分析哈
```
* 包含特殊字符处理
```cpp
sscanf("201*1b_-cdZA&", "%[0-9|_|--|a-z|A-Z|&|*]", buf);
// 结果为：201*1b_-cdZA&
```

如果能将上述几个例子搞明白，相信基本上已经掌握了 sscanf 的用法，实践才是检验真理的唯一标准，只有多使用，多思考才能真正理解它的用法。

### fprintf/fscanf
fprintf 打印到指定的文件 stream 中，fscanf 从文件中格式化读取数据，类似 scanf 函数。相关函数的声明如下：
```cpp
int fprintf(FILE *stream, const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
```

还是通过简单实例来说明基本用法。
```cpp
FILE *fp = fopen("file.txt", "w");
fprintf(fp, "%d-%s-%f\n", 32, "hello", 0.12);
fclose(fp);
```
```sh
liwei:/tmp$ cat file.txt 
32-hello-0.120000
```

而 fscanf 函数的使用基本上与 sscanf 函数使用方式相同。

## 八、IO缓冲区
还有个关于 IO 非常重要的概念，就是 IO 缓冲区。

C 标准库为每个打开的文件分配一个 I/O 缓冲区，用户调用读写函数大多数都在 I/O 缓冲区中读写，只有少数请求传递给内核。

以 fgetc／fputc 为例，当第一次调用 fgetc 读一个字节时，fgetc 函数可能通过系统调用进入内核读 1k 字节到缓冲区，然后返回缓冲区中第一个字节给用户，以后用户再调用 fgetc，就直接从缓冲区读取。

另一方面，fputc 通常只是写到缓冲区中，如果缓冲区满了，fputc 就通过系统调用把缓冲区数据传递给内核，内核将数据写回磁盘。如果希望把缓冲区数据立即写入磁盘，可以调用 fflush 函数。

C 标准库 IO 缓冲区有三种类型：全缓冲、行缓冲和无缓冲区，不同类型的缓冲区具有不同的特性。
* `全缓冲`：如果缓冲区写满了就写回内核。常规文件通常是全缓冲的。
* `行缓冲`：如果程序写的数据中有换行符就把这一行写回内核，或者缓冲区满就写回内核。标准输入和标准输出对应终端设备时通常是行缓冲的。
* `无缓冲`：用户程序每次调用库函数做写操作都要通过系统调用写回内核。标准错误输出通常是无缓冲的，用户程序的错误信息可以尽快输出到设备。

```cpp
printf("hello world");
while(1);
// 运行程序会发现屏幕并没有打印hello world
// 因为缓冲区没满，且没有\n符号
```

除了写满缓冲区、写入换行符之外，行缓冲还有一种情况会自动做 flush 操作，如果：
* 用户程序调用库函数从无缓冲的文件中读取
* 或从行缓冲的文件中读取，且这次读操作会引发系统调用从内核读取数据，那么会读之前自动 flush 所有行缓冲
* 程序退出时通常也会自动 flush 缓冲区

如果不想完全依赖自动的 flush 操作，可以调用 fflush 函数手动操作。若调用 fflush(NULL) 可以对所有打开文件的 IO 缓冲区做 flush 操作。缓冲区大小也可以自定义设置，一般情况无需设置，默认即可。

