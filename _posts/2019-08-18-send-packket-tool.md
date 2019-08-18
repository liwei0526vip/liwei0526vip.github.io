---
layout: post
date: "2019-08-18 11:20:33"
title:  "常用的发包工具使用总结"
categories: Linux工具
tags: 发包
author: 肖邦
---

* content
{:toc}

> 对于底层或后端开发的程序人员来说，通常会使用一些发包工具，来实现报文级别的功能测试以及性能压力测试，所以总结出好用的、切合实际业务场景的发包工具非常有必要。




## 常用的发包工具

* wrk [Github 地址](https://github.com/wg/wrk)
* http_load [官方地址](http://www.acme.com/software/http_load/)
* ab（[apache bench](https://httpd.apache.org/docs/2.4/programs/ab.html)）
* webbench [官方地址](http://home.tiscali.cz/~cz210552/webbench.html)

## 发包工具 wrk

wrk 是一个简单易用、高性能的 http 发包测试工具，同时支持 IPv6、https 。另外支持自定义的 Lua 脚本，可以自定义请求生成、响应处理、自定义报告内容等，在 Github 上已经有 2 万多个 Star 了。

* 安装
    ```bash
    $ cd $workdir
    $ git clone https://github.com/wg/wrk.git
    $ cd $workdir/wrk
    $ make -j12  # 使用 12 个线程进行编译
    $ sysctl -w tcp_max_tw_buckets=0
    # 设置 tcp_max_tw_buckets 取消 time_wait 队列，为了快速复用端口以满足高并发连接
    ```

* 基本用法
    ```bash
    $ ./wrk -t12 -c400 -d30s http://127.0.0.1:8080/index.html
    # -t: 总的线程数，一般设置为 cpu 个数
    # -c: 每个线程保持的连接数，一般设置为线程的 20 倍或 30 倍，使情况而定。
    # -d: 测试运行的时间，支持格式：2s，2m，2h
    # 其它选项含义参考：https://github.com/wg/wrk#command-line-options
    ```

* 输出信息
    ```bash
    Running 30s test @ http://127.0.0.1:8080/index.html
      12 threads and 400 connections
      Thread Stats   Avg      Stdev     Max   +/- Stdev
        Latency   635.91us    0.89ms  12.92ms   93.69%
        Req/Sec    56.20k     8.07k   62.00k    86.54%
      22464657 requests in 30.00s, 17.76GB read
    Requests/sec: 748868.53
    Transfer/sec:    606.33MB
    ```

## 发包工具 http_load

* http_load 安装
    ```bash
    $ wget http://www.acme.com/software/http_load/http_load-09Mar2016.tar.gz
    $ tar zxf http_load-09Mar2016.tar.gz
    $ make
    ```

* 基本用法
    ```bash
    # 封装到 shell 脚本中
    
    #!/bin/bash
    
    if [[ $# -lt 2 ]]; then
        echo "usage: ./http_load_test <parallel number> <times>"
        exit 1
    fi
    
    rm -f ./results/result*
    
    for (( i=1; i<=$1; i ++ )); do
        echo "http_load process$i runs in CPU$i"
        /bin/taskset -c $i ../http_load-09Mar2016/http_load -sip ./sip$i.txt -p 50 -s $2 ./url.txt > ./results/result$i &
        sleep 2
    done
    ```

* 输出数据（存储在 result 目录中的 resultN 文件中）

* 注意几个参数
    ```bash
    -sip : 指定源 IP 文件（sip1.txt 文件中记录的源 IP 地址）
    -p : 并发连接数
    -s : 运行时间
    ```

## 发包工具 webbench

* webbench 安装
    ```bash
    $ wget http://home.tiscali.cz/~cz210552/distfiles/webbench-1.5.tar.gz
    $ tar zxvf webbench-1.5.tar.gz
    $ cd webbench-1.5
    $ make
    $ make install
    ```

* 基本用法
    ```bash
    $ webbench -c 100 -t 100 http://192.168.11.15:8088/
    # -c : 客户端的个数，webbench 通过 fork 出 N 个进程来模拟客户端
    # -t : 测试持续的时间，单位为：秒
    ```

* 输出数据
    ```bash
    Benchmarking: GET http://192.168.11.15:8088/
    200 clients, running 10 sec.
    
    Speed=25383546 pages/min, 24114374 bytes/sec.
    Requests: 4230591 susceed, 0 failed.
    ```

## 发包工具 ab

ab 是 apache 服务器自带的一个压力测试工具，如果想单独安装，可以执行如下命令：

```bash
$ yum install httpd-tools -y
```

* 安装完成之后，就可以使用 ab 进行测试：
    ```bash
    $ ab -n 1000 -c 100 https://baidu.com
    # -n : 本次测试执行 N 个请求数
    # -c : 指的是并发连接数
    ```

* 输出数据
    ```bash
    Concurrency Level:      100
    Time taken for tests:   25.404 seconds
    Complete requests:      1000000
    Failed requests:        0
    Write errors:           0
    Total transferred:      57000000 bytes
    HTML transferred:       0 bytes
    Requests per second:    39363.39 [#/sec] (mean)
    Time per request:       2.540 [ms] (mean)
    Time per request:       0.025 [ms] (mean, across all concurrent requests)
    Transfer rate:          2191.13 [Kbytes/sec] received
    Connection Times (ms)
                  min  mean[+/-sd] median   max
    Connect:        0    1   0.2      1       2
    Processing:     0    1   0.2      1       3
    Waiting:        0    1   0.2      1       3
    Total:          1    3   0.1      3       4
    ... ...
    ```

测试结果中，重点关注 `Requests per second` 和 `Time per request` ，分别是每秒请求数和单个请求耗时。



## dpdk-pktgen

待更新。。。

