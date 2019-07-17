---
layout: post
date: "2019-06-03 18:31:30"
title:  "Openresty之HTTP服务的功能接口"
categories: OpenResty
tags: OpenResty
author: 肖邦
---

* content
{:toc}

> * 基于高效的 Nginx 平台和小巧紧凑的 Lua 语言，我们可以在 openresty 里以脚本编程的方式轻易的构建出高性能的 HTTP 服务，实现 Web 容器和 RESTFUL 应用架构.
> * Openresty 完美的结合了 Nginx 的事件驱动机制和 Lua 的协程机制，所有的函数都是 `同步非阻塞` 的，处理请求时不需要像其他语言那样编写出难以理解的异步回调函数，自然而高效。
> * Openresty 最初目标就是要 `方便快捷` 开发 HTTP 服务。




## 一、简介

Openresty 基于 Nginx 可以任意操作请求行、请求头、请求体、响应头、响应体，也支持 chunked、keepalive、lingering_close 等特性。开发 HTTP 服务主要用到的执行阶段有：

* `set_by_lua` ：改写 Nginx 变量，相当于 set；
* `rewrite_by_lua` ： 改写 URI，可用于实现跳转/重定向；
* `access_by_lua` ： 多用于处理访问控制或限速；
* `content_by_lua` ：最常用的阶段，产生响应内容；
* `header_filter_by_lua` ：加工处理响应头，过滤数据；
* `body_filter_by_lua` ：加工处理响应体，可附加额外内容；
* `log_by_lua` ：记录日志、统计分析或其他的首尾工作。


## 二、配置指令

以下的指令可以用在配置文件的 `http{}` 中调整 Openresty 处理 HTTP 请求时的行为。

* `lua_use_default_type on | off`。在发送响应数据时是否在响应头字段 `Content-Type` 里使用默认的 MIME 类型，通常应设置为 on
* `lua_malloc_trim num`。设置清理内存的周期，参数 num 是请求的次数。当处理了 num 个请求后就会把进程内空闲内存归还给系统，最小化内存占用。参数 num 默认值为 1000，如果系统的内存足够大，那么可以设置为 0，可以禁用内存清理。
* `lua_need_request_body on | off`。是否要求 Openresty 在开始处理流程前强制读取请求体数据，默认为 off，不会主动读取请求体。
* `lua_http10_buffering on | off`。启用或禁用 HTTP 1.0 里的缓冲机制，默认是 on。该指令仅为了兼容 HTTP 1.0/0.9 协议，目前 HTTP1.1 基本已经全面普及，建议把它设置为 off，可以加快 Openresty 的处理速度。
* `lua_check_client_abort on | off`。是否启用 Openresty 的客户端意外断连检测，默认是 off。如果打开此指令，需要在 lua 程序中编写一个 handler 来处理断连。


## 三、常量

**状态码表示 HTTP 请求的处理状态**

* `ngx.HTTP_OK` ：200，请求已成功处理；
* `ngx.HTTP_MOVED_TEMPORARILY` ：302，重定向跳转；
* `ngx.HTTP_BAD_REQUEST` ：400，客户端请求错误；
* `ngx.HTTP_UNAUTHORIZED` ： 401，未认证；
* `ngx.HTTP_FORBIDDEN` ：403，禁止访问；
* `ngx.HTTP_NOT_FOUND` ：404，资源未找到；
* `ngx.HTTP_INTERNAL_SERVER_ERROR` ：500，服务器内部错误；
* `ngx.HTTP_BAD_GATEWAY` ：502，网关错误，反向代理后端无效响应；
* `ngx.HTTP_SERVICE_UNAVAILABLE` ：503，服务器暂不可用；
* `ngx.HTTP_GATEWAY_TIMEOUT` ：504，网关超时，反向代理时后端超时；

**请求方法**

* `ngx.HTTP_GET` ：读操作，读取数据；
* `ngx.HTTP_HEAD` ： 读操作，获取元数据；
* `ngx.HTTP_POST` ：写操作，提交数据；
* `ngx.HTTP_PUT` ：写操作，更新数据；
* `ngx.HTTP_DELETE` ：写操作，删除数据；
* `ngx.PATCH` ：写操作，局部更新数据。


## 四、变量

Openresty 使用表 `ngx.var` 操作 Nginx 变量，里面包含了所有的内置变量和自定义变量，可以用名字直接访问。例如，请求地址、请求参数、请求头、客户端地址、收发字节数等。

**1、读变量**

```lua
ngx.say(ngx.var.uri)                                -- 输出变量 $uri，请求的 URI
ngx.say(ngx.var['http_host'])                       -- 输出变量 $http_host，请求头里的 host
assert(not ngx.var.xxx)                             -- 变量 $xxx 不存在，所以是 nil

if #ngx.var.is_args > 0 then                        -- 检查是否有 URI 参数
    ngx.say(ngx.var.args)                           -- 输出 URI 里的参数
end

local str = "$http_host:$server_port"               -- 一个包含多个变量的字符串

str = ngx.re.gsub(str, [[\$(\w+)]], function(m),    -- 使用正则替换，替换时使用函数操作
    return ngx.var[m[1]] or "" end, "jo")

type(ngx.var.request_length)                        -- $request_length 类型是 string
```

当心所有的变量类型都是字符串，即使它表现为数字。在 Lua 代码里如果要把它们与数字作运算必须调用 `tonumber` 转换。`ngx.var.xxx` 虽然很方便，不过每次调用会有一些额外开销，但不建议多度使用，应当尽量使用 Openresty 等价的功能接口，如果必须要使用则最好 `local` 化暂存，避免多次调用。
```lua
local uri = ngx.var.uri                 -- local 化，避免多次调用 ngx.var
```


**2、写变量**

Nginx 内置的变量绝大多数是只读的，只有 `$args` `$limit_rate` 等少数可以直接修改。
```lua
ngx.var.limit_rate = 1024 * 2           -- 改写限速变量为 2k
ngx.var.uri = "unchangeable"            -- 不可修改，会在运行日志里记录错误信息
```

配置文件里使用 `set` 指令自定义的变量是可写的，允许任意赋值修改，由于变量在处理流程中全局可见，我们可以利用它来在请求处理的各个阶段之间传递数据。

`set_by_lua` 是另一种改写变量的方式，它类似指令 set 或 map，但能够使用 Lua 代码编写复杂的逻辑赋值给变量：
```lua
set_by_lua_block $var {
    local len = ngx.var.request_length
    return tonumber(len) * 2
}
```
不过不建议使用 `set_by_lua`，它的限制较多，阻塞操作，一次只能赋值一个变量，不能使用 `cosocket` 等，


## 五、基本信息

通常我们的程序是从 Rewrite 阶段才开始处理请求，而在之前的 Preread 阶段 Openresty 已经从客户端获取了一些基本的信息，包括来源、起始时间和请求头文本。

**请求来源**

函数 `ngx.req.is_internal()` 用来判断本次请求是否由 `外部` 发起的，如果是内部的流程跳转或子请求，那么函数的返回值就是 `true`。

**起始时间**

函数 `ngx.req.start_time()` 可以获取服务器开始处理本次请求时的时间戳，精确到毫秒，使用它可以随时计算出请求的处理时间，相当于 `$request_time` 但更廉价。

**请求头**

函数 `ngx.req.raw_header()` 可以获取 HTTP 请求头的原始文本，利用正则可以解析字符串从中提取多种信息，不过使用 Openresty 提供的专用功能接口会更好。

**暂存数据**

Openresty 每个阶段执行的程序彼此独立，如果想要在各阶段间传递数据就需要使用 `ngx.ctx` 。Openresty 为每个 HTTP 请求都提供了一个单独的表 `ngx.ctx`，在整个处理流程中共享，可以用来保存任意的数据或计算的中间结果。
```lua
rewrite_by_lua_block {
    local len = ngx.var.content_length
    ngx.ctx.len = tonumber(len)
}
access_by_lua_block {
    assert(not len)
    assert(ngx.ctx.len)
}
content_by_lua_block {
    ngx.say(ngx.ctx.len)                -- ngx.ctx 里的变量在其它阶段仍然可用
}
```
但 `ngx.ctx` 的成本较高，应当尽量少用，只存放少量必要的数据，避免滥用。


## 六、请求行

HTTP 请求行的信息包含：请求方法、URI、HTTP 版本等，可以用 ngx.var 获取，例如：
* `$request` ：完整的请求行
* `$scheme` ：协议的名字，如 `http` 或 `https`
* `$request_method` ：请求的方法
* `$request_uri` ：请求的完整 URI
* `$uri` ：请求的地址，不包含 `？` 及后面的参数
* `$document_uri` ：同 `$uri`
* `$args` ：URI 里的参数，即 `？` 后的字符串
* `$arg_xxx` ：URI 里名为 xxx 的参数值

因为 `ngx.var` 的方式效率不高，而且是只读的，OpenResty 在表 `ngx.req` 里提供了个专门操作请求行的函数。这些函数多用在 `rewrite_by_lua` 阶段，改写 URI 的各种参数，实现重定向跳转。

* `ngx.req.http_version()` 以数字形式返回请求行里的 HTTP 协议版本号，目前可能的值是 `0.9 1.0 1.1 2`
* `ngx.req.get_method()` 和 `ngx.req.set_method()`，可以读写当前的请求方法，但需要注意的是，前者返回值是字符串，而后者的参数却不能用字符串，只能用数字常量。
* `ngx.req.set_uri()` 用于改写请求行里的地址部分，只能在 `rewrite_by_lua` 阶段里把它设置为 true，将执行一个内部重定向，跳转到本 server 内其他 location 里继续处理请求。
  ```lua
  ngx.req.set_uri(uri, jump)                -- 改写请求行的uri，jump 默认值为 false
  local uri = "a + b = c #!"                -- 待编码的字符串
  local enc = ngx.escape_uri(uri)           -- 转义其中的特殊字符
  local dec = ngx.unescape_uri(enc)         -- 还原字符串
  ```
* `ngx.req.get_uri_args()`，获取 URI 里的参数，以 `key-value` 表的形式返回结果。该函数会解析所有参数，当参数很多而只用其中几个参数时成本显得较高，建议直接使用 `ngx.var.arg_xxx`
* `ngx.req.get_post_args()`，使用前必须先调用 `ngx.req.read_body()` 读取数据。
* `ngx.req.set_uri_args()`，改写 URI 里的参数部分，可以接受两种形式的参数，第 1 种是标准的 URI 字符串，第 2 种是 Lua 表。
* `参数编码`
  ```lua
  local args = { n = 1, v = 100 }
  local enc = ngx.encode_args(args)         -- 编码，结果是 "v=100&n=1"
  local dec = ngx.decode_args(enc)
  ```


## 七、请求头

**读取数据**

函数 `ngx.req.get_headers()` 解析所有的请求头，返回一个表：
```lua
local headers = ngx.req.get_headers()
```
为了能够在 Lua 代码里作为名字使用，头字段在解析后有了两点变化：
```lua
ngx.say(headers.host)                   -- Host
ngx.say(headers.user_agent)             -- User-Agent
```
不过 `[]` 方式允许使用字段的原始形式：
```lua
ngx.say(headers['User-Agent'])
ngx.say(headers['Accept'])
```
同理，如果只想读取其中的少数字段时建议直接使用 `ngx.var.http_xxx`

**改写数据**

```lua
ngx.req.set_header("Accept", "Firefox")         -- 改写头字段 Accept
ngx.req.set_header("Metroid", "Prime 4")        -- 新增头字段 Metroid
ngx.req.set_header("Metroid", nil)              -- 使用 nil 删除头字段 Metroid
ngx.req.clear_header("Accept")                  -- 删除头字段 Accept
```

## 八、请求体

请求体是 HTTP 请求头之后的数据，通常由 POST 或 PUT 方法发送，可以从客户端得到大块的数据。

**1、丢弃数据**

```lua
ngx.req.discard_body()          -- 显式丢弃请求体
```

**2、读取数据**

出于效率考虑，Openresty 不会主动读取请求体数据，除非用指令 `lua_need_request_body on`，读取请求体需要执行下面的步骤：
* 调用函数 `ngx.req.read_body` ，开始读取请求体数据
* 调用函数 `ngx.req.get_body_data` 获取数据，相当于 `$request_body`
* 如果得到是 `nil`，可能是数据过大，存放在了磁盘文件里，调用函数 `ngx.req.get_body_file` 可以获取相应的临时文件名
* 使用 `io.*` 函数打开文件，读取数据(注意是阻塞操作)
```lua
ngx.req.read_body()
local data = ngx.req.get_body_data()
if data then
    ngx.say("body: ", data)
else
    local name = ngx.req.get_body_file()
    local f = io.open(name, "r")
    data = f:read("*a")
    f:close()
end
```
在 Openresty 中请求体超过 `16KB` ，超过此值就会存到硬盘上，可以通过指令 `client_body_buffer_size` 调整。

**3、改写数据**

可以通过 `ngx.req.set_body_data` 或 `ngx.req.set_body_file` 来改写请求体。也可以通过下面函数逐步创建一个新的请求体：
```lua
ngx.req.init_body()
ngx.req.append_body('aaa')
ngx.req.append_body('bbb')
ngx.req.finish_body()
```


## 九、响应头

HTTP 协议的响应头包括状态行和响应头字段，Openresty 会设置它们的默认值。

**1、改写数据**

```lua
ngx.status = ngx.HTTP_ACCEPTED
ngx.header['Server'] = 'my openresty'
ngx.header.content_length = 0
ngx.header.new_field = 'xxx'
ngx.haader.date = nil                           -- 删除 date 字段
```
函数 `ngx.resp.add_header()` 可以新增头字段，但它不会覆盖同名的字段。

**2、发送数据**

调用函数 `ngx.send_headers` 可以显式地发送响应头。

**3、过滤数据**

响应头数据在发送到客户端的途中经过 Openresty 的 filter 阶段，即 `header_filter_by_lua` 在这里可以改写状态码和头字段。
```lua
header_filter_by_lua_block {
    if ngx.header.etag then
        ngx.header.etag = nil
    end
    ngx.header["Cache-Control"] = "max-age=300"
}
```
与 `content_by_lua` 相比两者区别是所在的执行阶段，`header_filter_by_lua` 是数据传输的中间点，用来修改数据，而 `content_by_lua` 是数据的起点、来源，用来生产数据。而且在不能使用 `content_by_lua` 情况下(proxy_pass)，`header_filter_by_lua` 更是改写响应数据的唯一手段。


## 十、响应体

在 Openresty 中发送响应体不需要考虑缓冲、异步、回调、分块的问题，Openresty 会自动处理这一切。

**发送数据**

`ngx.print` `ngx.say`

**过滤数据**

`body_filter_by_lua` 对响应数据作任意的修改、删除或截断，改写客户端最终收到的数据：
```lua
body_filter_by_lua_block {
    if ngx.re.find (ngx.arg[1], 'xxx', "jo") then
        ngx.arg[1] = nil
        return
    end
    if ngx.arg[2] then
        ngx.arg[1] = ngx.arg[1]..'xx'
    end
}
```


