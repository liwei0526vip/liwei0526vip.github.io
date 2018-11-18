---
layout: post
title:  "Nginx虚拟主机配置详解"
categories: Nginx
tags: Nginx
author: 肖邦
---

* content
{:toc}

> 在 Nginx 中 http{} 中的 server{} 配置块是对服务器的配置，Nginx 支持配置多个 server{}，每个 server 配置是一个虚拟主机，也就是一个站点。合理配置虚拟主机是网站能够正常运行的前提条件。




## 配置一个虚拟主机

通常情况，如果配置多个虚拟主机的话，我们不直接在 Nginx 主配置文件 nginx.conf 进行配置，避免 nginx.conf 配置文件大而复杂，不易维护。Nginx 支持 `include` 的配置方式，我们可以将多个 server 配置文件分开放到 vhost 目录中，然后在 nginx.conf 中 `include vhost/*.conf` 即可。

配置一个域名为 "www.foo.com" 虚拟主机。先删除 nginx.conf 配置文件中 server 配置，然后在 nginx.conf 配置的 http{} 最下方添加 `include vhost/*.conf;`
```bash
$ cat conf/vhost/www.foo.com.conf
server {
    listen 80;
    server_name www.foo.com;
    root /data/www/www.foo.com;
    index index.html;
}
```

重新加载配置文件，测试：
```bash
$ nginx -t
$ nginx -s reload
$ curl -x127.0.0.1:80 www.foo.com
  www.foo.com
```

## 配置第2个虚拟主机

再添加一个 "www.bar.com" 的虚拟主机
```bash
$ cat conf/vhost/www.bar.com.conf
server {
    listen 80;
    server_name www.bar.com;
    root /data/www/www.bar.com;
    index index.html;
}
```

重新加载配置文件，测试：
```bash
$ nginx -t
$ nginx -s reload
$ curl -x127.0.0.1:80 www.bar.com
  www.bar.com
```


## 默认虚拟主机

我们配置了两个虚拟主机："www.foo.com" 和 "www.bar.com"，当我们访问 foo 和 bar 的域名时，分别访问对应的站点内容。当我们访问 "www.1.com" 时，结果返回 "www.bar.com" 的内容，为何？这是因为 Nginx 默认已配置的第 1 个虚拟主机为 `默认虚拟主机` ，当访问的域名没有配置时，会命中 `默认虚拟主机`，也就是 "bar" 这个虚拟主机。

* 关于第 1 个虚拟主机。因为 Nginx 加载配置时按照名称排序，"www.bar.com" 在 "www.foo.com" 前边，所以，Nginx 认为 "bar" 是配置的第一个虚拟主机，即默认虚拟主机。

* 设置默认虚拟主机必要性。正常情况下我们都要设置默认虚拟主机，这样的话，通过 IP 地址和其他域名访问的请求，Nginx 会命中到默认虚拟主机，我们通常会将默认虚拟主机设置为 `deny`，只有正常配置的域名才能访问我们的网站。

* 使用 default_server 设置默认虚拟主机。Nginx 默认是按照名称顺序来识别第 1 个虚拟主机为默认虚拟主机的，如果配置虚拟主机较多的话，容易出错。我们需要显式地指定默认虚拟主机。

  ```bash
  $ cat conf/vhost/default.conf
  server {
      listen 80 default_server;
      deny all;
  }

  $ nginx -s reload
  $ curl -x127.0.0.1:80 www.baidu.com
    ...
    <center><h1>403 Forbidden</h1></center>
    ...
  ```
 

## 泛域名解析配置

有时候我们会将多个二级域名都解析到一个虚拟主机上，可以使用泛解析的方式。

```bash
$ cat conf/vhost/linuxblogs.cn.conf
  server {
      listen 80;
      server_name *.linuxblogs.cn;
      root /data/www/linuxblogs.cn;
  }
```
重新加载配置，进行测试：
```bash
$ nginx -s reload
$ curl -x127.0.0.1:80 a.linuxblogs.cn
  www.linuxblogs.cn
$ curl -x127.0.0.1:80 b.linuxblogs.cn
  www.linuxblogs.cn
$ curl -x127.0.0.1:80 linuxblogs.cn
  403 Forbidden    # 不能匹配 linuxblogs.cn
```
如果需要匹配 `linuxblogs.cn` 域名，需要在 server_name 后加上本域名。

```
server_name linuxblogs.cn *.linuxblogs.cn
```


## 配置基于端口的虚拟主机

Nginx 可以配置基于端口的虚拟主机。前边我们已经配置了 "www.foo.com" 的 80 端口的虚拟主机，我们可以再配置一个 8080 端口的虚拟主机。

```bash
$ cat www.foo.com-8080.conf 
  server {
      listen 8080;
      server_name www.foo.com;
      root /data/www/www.foo.com;
  }
$ nginx -s reload
$ curl -x127.0.0.1:8080 www.foo.com
  www.foo.com
```

