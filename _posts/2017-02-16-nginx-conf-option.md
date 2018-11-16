---
layout: post
title:  "Nginx常用配置参数详解"
categories: Nginx
tags: Nginx
author: 肖邦
---

* content
{:toc}

> Nginx 配置文件中常用参数的含义总结。

## 全局配置

* user nobody;

  定义运行 nginx 服务进程的用户，还可以加上组，如：user nobody nobody;
* worker_processes 1;
  
  定义 worker 进程的数量，一般设置和 CPU 核数保持一致。还可以设置为 auto，表示让系统自动调整。
* error_log logs/error.log;

  定义错误日志路径，也可以定义到 http、server、location 里。
* error_log logs/error.log notice;

  定义错误日志级别，级别越高记录信息越少，默认是error，常见的错误日志级别有：
  ```
  [debug|info|notice|warn|error|crit|alert|emerg]
  ```
* pid logs/nginx.pid;

  定义nginx进程pid文件所在路径。
* worker_rlimit_nofile 100000;

  定义nginx最多打开文件数限制。如果没设置的话，这个值为操作系统(ulimit -n)的限制保持一致。把这个值设高，nginx就不会有 "too many open files" 问题了。


## events 配置部分

* worker_connections 1024;

  定义每个 work_process 同时开启的最大连接数，即允许最多只能有这么多连接。
* accept_mutex on;

  当某一个时刻只有一个网络连接请求服务器时，服务器上有多个睡眠的进程会被同时叫醒，这样会损耗一定的服务器性能。Nginx 中的 accept_mutex 设置为 on，将会对多个 Nginx 进程(worker processer)接收连接时进行序列化，防止多个进程争抢资源。

* multi_accept on;

  nginx worker processer 可以做到同时接收多个新到达的网络连接，前提是把该参数设置为 on，默认为 off。

* use epoll;

  Nginx 服务器提供了多个事件驱动模型来处理网络消息，其支持的类型有：select poll kqueue epoll rtsing /dev/poll eventport。Linux 服务器上默认是 epoll。


## HTTP 配置部分

* 参考链接

  - [nginx http模块配置参数解读](https://segmentfault.com/a/1190000012672431)
  - [nginx服务器安装及配置文件详解](https://segmentfault.com/a/1190000002797601)
  - [HTTP Header 详解](https://kb.cnblogs.com/page/92320/)

* MIME-Type

  ```
  include mime.tpyes;    # 定义 nginx 能识别的网络资源媒体类型，如，文本、html、js 等
  default_type application/octet-stream;  # 如果不定义该行，默认为 text/plain
  ```

* log_format

  ```
  log_format lb '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
  # 其中 lb 为日志格式的名字，定义日志路径时会引用 lb 这个日志名称。
  ```

* access_log logs/access.log lb;

  定义日志路径以及采用的格式，该参数可以在 server 配置块中定义。

* sendfile on;

  调用 sendfile 函数传输文件，默认为 off。使用 sendfile 可以减少用户空间和内核空间的切换和数据拷贝，从而提升服务器性能。对于普通应用设置为 on，如果用来进行下载等应用磁盘 IO 重负载应用，可设置为 off，以平衡磁盘与网络 IO 处理速度，降低系统的负载。

* sendfile_max_chunk 128k;

  该参数限定 Nginx worker process 每次调用 sendfile 函数传输数据的最大值，默认为 0，表示无限制。

* tcp_nopush on;

  当 tcp_nopush 设置为 on 时，会调用 tcp_cork 方法进行数据传输。具体作用是：当应用程序产生数据时，内核不会立马封装包，而是当数据量积累到一定量时才会封装，然后传输，这样有助于解决网络堵塞问题，默认值为 on。

* keepalived_timeout 65 60;

  该参数有两个值。第 1 个值设置 nginx 服务器与客户端会话结束后仍旧保持连接的最长时间，单位是秒，默认为 75 秒。第 2 个值可以省略，是针对客户端的浏览器来设置的，其实就是响应 HTTP 报文头中 keep-alive 选项，设置后，浏览器就会根据这个数值决定何时主动关闭连接，Nginx 服务器就不操心了，但有的浏览器并不认可该参数。

* send_timeout;

  这个超时时间是发送响应的超时时间，即 Nginx 服务器向客户端发送了数据包，但客户端一直没有去接收这个数据包。如果某个连接超过 send_timeout 定义的超时时间，那么 Nginx 将会关闭这个连接。

* client_max_body_size 10m;

  浏览器在发送较大 HTTP 包体的请求时，头部会有一个 Content-Length 字段，该参数就是用来限制 Content-Length 所示值的大小的。比如用户上传一个 1GB 的文件，Nginx 在收完包头后发现超过参数定义的值，就直接发送 413 响应给客户端。

* gzip on;

  是否开启 gzip 压缩功能。

* gzip_min_length 1k;

  设置允许压缩的页面最小字节数，页面字节数从 header 头的 content-length 中获取。默认值是 20。建议设置成大于 1k 的字节数，小于 1k 可能会越压越大。

* gzip_buffers 4 16k;

  设置系统获取 4 个 16k 的 buffer 用于存储 gzip 的压缩结果数据流。

* gzip_comp_level 6;

  gzip 压缩比，1 压缩比最小处理速度对快，9 压缩比最大但处理速度最慢(传输快但比较消耗CPU)

* gzip_types mime-type;

  匹配 mime 类型进行压缩，无论是否指定，"text/html" 类型总是会被压缩的。比如，gzip_types text/plain text/css text/html

* gzip_proxied any;

  Nginx 作为反向代理的时候启用，决定开启或关闭后端服务器返回的结果是否压缩，前提是后端服务器必须要包含 "Via" 的头。

* gzip_vary on;

  和 http 头有关系，会在响应头加个 vary：Accept-Encoding，可以让前端的缓存服务器缓存经过 gzip 压缩的页面。


## server 部分配置

server 包含在 http{} 内部，每一个 server{} 都是一个虚拟主机(站点)。

```bash
server {
    listen       127.0.0.1:80;
    server_name  localhost www.linuxblogs.cn;
    location / {
        root html;
        index index.html index.htm;
    }
    error_page  500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }
}
```

... 待更新 ...