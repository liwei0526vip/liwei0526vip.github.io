---
layout: post
title:  "深入理解 Linux 的守护进程"
categories: Linux系统
tags: 守护进程 Linux
author: 肖邦
---

* content
{:toc}

> 「守护进程」是 Linux 的一种长期运行的后台服务进程，也有人称它为「精灵进程」。我们常见的 httpd、named、sshd 等服务都是以守护进程 Daemon 方式运行的，通常服务名称以字母ｄ结尾，也就是 Daemon 第一个字母。与普通进程相比它大概有如下特点：




- 无需控制终端(不需要与用户交互)
- 在后台运行
- 生命周期比较长，一般是随系统启动和关闭

## 守护进程必要性
为什么要设置为守护进程，普通进程不可以吗？

当我们在命令行提示符后输入类似`./helloworld`程序时，在程序运行时终端被占用，此时无法执行其它操作。即使使用`./helloworld &`方式后台运行，当连接终端的网络出现问题，那么也会导致运行程序中断。这些因素对于长期运行的服务来说很不友好，而「守护进程」可以很好的解决这个问题。

## 对进程组、会话、终端的理解
「守护进程」理解起来并不复杂，代码编写上有基本固定的套路。如果想要深入理解「守护进程」基本原理，那么必须要首先理解 Linux 的进程、进程组、会话、终端等概念。

### 1、进程
- 进程是 Linux 进行资源分配的最小单位
- 前台进程，例如这样：`$ ./hello`
- 后台进程，例如这样：`$ ./hello &` 释放对控制终端的占用

### 2、进程组
每个进程都会属于一个进程组，进程组中可以包含一个或多个进程。进程组中有一个进程组长，组长的进程 ID 是进程组 ID(PGID)
```bash
$ ps -o pid,pgid,ppid,comm | cat
  PID  PGID  PPID  COMMAND
10179  10179 10177 bash
10263  10263 10179 ps
10264  10263 10179 cat
```
下边通过简单的示例来理解进程组
- bash：进程和进程组ID都是 10179，父进程其实是 sshd(10177)
- ps：进程和进程组ID都是 10263，父进程是 bash(10179)，因为是在 Shell 上执行的命令
- cat：进程组 ID 与 ps 的进程组 ID 相同，父进程同样是 bash(10179)

容易理解 Bash 就是Shell进程，Shell 父进程是 sshd；ps 与 cat 通过管道符号一起运行，属于一个进程组，其父进程都是 Bash；一个进程组也被称为「作业」。

### 3、会话(session)
多个进程组构成一个「会话」，建立会话的进程是会话的领导进程，该进程 ID 为会话的 SID。会话中的每个进程组称为一个「作业」。会话可以有一个进程组称为会话的「前台作业」，其它进程组为「后台作业」

一个会话可以有一个控制终端，当控制终端有输入和输出时都会传递给前台进程组，比如`Ctrl + Z`。会话的意义在于能将多个作业通过一个终端控制，一个前台操作，其它后台运行。

### 4、前后台作业相关操作
让作业由进入后台运行：
```bash
$ ping localhost >/dev/null &
[1] 10269    # 终端显示
# [1]：作业ID  10269：进程组ID
```
给后台作业发信号 SIGTERM
```bash
$ kill -SIGTERM -10269  # 发信号给进程组
$ kill -SIGTERM %1      # 发信号给作业1
```
让后台进程切换到前台：
```bash
$ fg %1
# ping 进程重新切到前台
```

## 编写守护进程
编写守护进程看似复杂，但实际上也是遵循一个特定的流程。
### 1、创建子进程，父进程退出
进程 fork 后，父进程退出。这么做的原因有 2 点：
- 如果守护进程是通过 Shell 启动，父进程退出，Shell 就会认为任务执行完毕，这时子进程由 init 收养
- 子进程继承父进程的进程组 ID，保证了子进程不是进程组组长，因为后边调用`setsid()`要求必须不是进程组长

### 2、子进程创建新会话
调用`setsid()`创建一个新的会话，并成为新会话组长。这个步骤主要是要与继承父进程的会话、进程组、终端脱离关系。

### 3、禁止子进程重新打开终端
此刻子进程是会话组长，为了防止子进程重新打开终端，再次 fork 后退出父进程，也就是此子进程。这时子进程 2 不再是会话组长，无法再打开终端。其实这一步骤不是必须的，不过加上这一步骤会显得更加严谨。

### 4、设置当前目录为根目录
如果守护进程的当前工作目录是`/usr/home`目录，那么管理员在卸载`/usr`分区时会报错的。为了避免这个问题，可以调用`chdir()`函数将工作目录设置为根目录`/`。

### 5、设置文件权限掩码
文件权限掩码是指屏蔽掉文件权限中的对应位。由于使用 `fork()`函数新建的子进程继承了父进程的文件权限掩码，这就给该子进程使用文件带来了诸多的麻烦。因此，把文件权限掩码设置为  0，可以大大增强该守护进程的灵活性。通常使用方法是`umask(0)`。

### 6、关闭文件描述符
子进程会继承已经打开的文件，它们占用系统资源，且可能导致所在文件系统无法卸载。此时守护进程与终端脱离，常说的输入、输出、错误描述符也应该关闭。


## 守护进程的出错处理
由于守护进程脱离了终端，不能将错误信息输出到控制终端，即使 gdb 也无法正常调试。常用的方法是使用 syslog 服务，将错误信息输入到`/var/log/messages`中。

syslog 是 Linux 中的系统日志管理服务，通过守护进程 syslogd 来维护。该守护进程在启动时会读一个配置文件`/etc/syslog.conf`。该文件决定了不同种类的消息会发送向何处。

## 守护进程编码示例
```c
pid_t pid, sid;
int i;
openlog("daemon_syslog", LOG_PID, LOG_DAEMON);
pid = fork(); // 第1步
if (pid < 0) exit(-1);
else if (pid > 0) exit(0);  // 父进程第一次退出
if ((sid = setsid()) < 0)   // 第2步
{
    syslog(LOG_ERR, "%s\n", "setsid");
    exit(-1);
}
// 第3步 第二次父进程退出
if ((pid = fork()) > 0) exit(0);
if ((sid = chdir("/")) < 0) // 第4步
{
    syslog(LOG_ERR, "%s\n", "chdir");
    exit(-1);
}
umask(0); // 第5步
// 第6步：关闭继承的文件描述符
for(i = 0; i < getdtablesize(); i++)
{
    close(i);
}
while(1)
{
    do_something();
}
closelog();
exit(0);
```

到这里基本上把守护进程的内容全部说清楚了，内容不少，概念比较晦涩，如果希望理解的比较透彻的话，可能需要多看几遍了。
