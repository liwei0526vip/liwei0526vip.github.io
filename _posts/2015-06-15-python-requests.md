---
layout: post
title:  "Python中requests使用总结"
categories: Python基础
tags: requests
author: 肖邦
---

* content
{:toc}

> requests 库可以实现 HTTP 协议中绝大部分功能，它提供的功能包括：keep-alive、连接池、Cookie 持久化、内容自动解压、HTTP 代理、SSL 认证、连接超时、Session 等很多特性，最重要的是它同时兼容 python2 和 python3，它是 Github 关注数最多的 Python 项目之一。




## 1、发送请求
```python
import requests

# 发送 GET 请求
response = requests.get("https://www.linuxblogs.cn")
```

## 2、响应内容
请求返回的值是一个 Response 对象，Response 对象是对 HTTP 协议中服务端返回给浏览器的响应数据的封装，响应中的主要元素包括：状态码、原因短语、响应首部、响应体等，这些属性都封装在 Response 对象中。

```python
# 状态码
>>> response.status_code
200

# 原因短语
>>> response.reason
'OK'

# 响应首部
for name,value in response.headers.items():
    print("%s:%s" % (name, value))
Server:nginx
Date:Mon, 24 Sep 2018 11:31:23 GMT
Content-Type:text/html; charset=utf-8
Content-Length:44078

# 响应内容
>>> response.content
```

requests 除了支持 GET 请求外，还支持 HTTP 规范中的其它所有方法，包括 POST、PUT、DELTET、HEADT、OPTIONS 方法。
```python
r = requests.post('http://httpbin.org/post', data = {'key': 'value'})
r = requests.put('http://httpbin.org/put', data = {'key': 'value'})
r = requests.delete('http://httpbin.org/delete')
r = requests.head('http://httpbin.org/get')
r = requests.options('http://httpbin.org/get')
```

## 3、查询参数
很多 URL 都带有很长一串参数，我们称这些参数为 URL 的查询参数，用 "?" 附加在 URL 链接后面，多个参数之间用 "&" 隔开，比如: http://fav.foofish.net/?p=4&s=20 ，现在你可以用字典来构建查询参数：
```python
args = {"p": 4, "s": 20}
response = requests.get("http://fav.foofish.net", params = args)
response.url
'http://fav.foofish.net/?p=4&s=2'
```

## 4、请求首部
requests 可以很简单地指定请求首部字段 Headers，比如有时要指定 User-Agent 伪装成浏览器发送请求，以此来蒙骗服务器。直接传递一个字典对象给参数 headers 即可。
```python
r = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})
```

## 5、请求体
requests 可以非常灵活地构建 POST 请求需要的数据，如果服务器要求发送的数据是表单数据，则可以指定关键字参数 data，如果要求传递 json 格式字符串参数，则可以使用json关键字参数，参数的值都可以字典的形式传过去。

* 作为表单数据传输给服务器
```python
payload = {'key1': 'value1', 'key2': 'value2'}
r = requests.post("http://httpbin.org/post", data=payload)
```
* 作为 json 格式的字符串格式传输给服务器
```python
import json
url = 'http://httpbin.org/post'
payload = {'some': 'data'}
r = requests.post(url, json=payload)
```


## 6、响应内容
HTTP返回的响应消息中很重要的一部分内容是响应体，响应体在 requests 中处理非常灵活，与响应体相关的属性有：content、text、json()。

* content 是 byte 类型，适合直接将内容保存到文件系统或者传输到网络中

```
r = requests.get("https://pic1.zhimg.com/v2-2e92ebadb4a967829dcd7d05908ccab0_b.jpg")
type(r.content)
<class 'bytes'>

# 另存为 test.jpg
with open("test.jpg", "wb") as f:
    f.write(r.content)
```

* text 是 str 类型，比如一个普通的 HTML 页面，需要对文本进一步分析时，使用 text。

```
r = requests.get("https://foofish.net/understand-http.html")
type(r.text)
<class 'str'>
re.compile('xxx').findall(r.text)
```

* 如果使用第三方开放平台或者API接口爬取数据时，返回的内容是json格式的数据时，那么可以直接使用json()方法返回一个经过json.loads()处理后的对象。

```
r = requests.get('https://www.v2ex.com/api/topics/hot.json')
r.json()
[{'id': 352833, 'title': '在长沙，父母同住...
```

## 7、代理设置
当爬虫频繁地对服务器进行抓取内容时，很容易被服务器屏蔽掉，因此要想继续顺利的进行爬取数据，使用代理是明智的选择。如果你想爬取墙外的数据，同样设置代理可以解决问题，requests 完美支持代理。

> 这里我用的是本地 ShadowSocks 的代理，(socks 协议的代理要这样安装 `pip install requests[socks]`)

```python
import requests
proxies = {
  'http': 'socks5://127.0.0.1:1080',
  'https': 'socks5://127.0.0.1:1080',
}
requests.get('https://www.linuxblogs.cn', proxies=proxies, timeout=5)
```

## 8、超时设置
requests 发送请求时，默认请求下线程一直阻塞，直到有响应返回才处理后面的逻辑。如果遇到服务器没有响应的情况时，问题就变得很严重了，它将导致整个应用程序一直处于阻塞状态而没法处理其他请求。正确的方式的是给每个请求显示地指定一个超时时间。

```python
r = requests.get("http://www.google.coma", timeout=5)
# 5秒后报错
Traceback (most recent call last):
socket.timeout: timed out
```

## 9、Session
* HTTP协议是一中无状态的协议，为了维持客户端与服务器之间的通信状态，使用 Cookie 技术使之保持双方的通信状态。
* 有些网页是需要登录才能进行爬虫操作的，而登录的原理就是浏览器首次通过用户名密码登录之后，服务器给客户端发送一个随机的Cookie，下次浏览器请求其它页面时，就把刚才的 cookie 随着请求一起发送给服务器，这样服务器就知道该用户已经是登录用户。

```python
import requests
# 构建会话
session  = requests.Session()
#　登录 url
session.post(login_url, data={username, password})
#　登录后才能访问的 url
r = session.get(home_url)
session.close()
```

构建一个 session 会话之后，客户端第一次发起请求登录账户，服务器自动把 cookie 信息保存在 session 对象中，发起第二次请求时 requests 自动把 session 中的 cookie 信息发送给服务器，使之保持通信状态。