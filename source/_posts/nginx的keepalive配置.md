---
title: nginx的keepalive配置
tags: 
	- nginx
	- keepalive
	- http
abstract: 本文讲述nginx有关keepalive的配置指令，相关的指令有keepalive、keepalive_requests、keepalive_timeout和keepalive_disable四个。它们主要作用是在配置nginx支持http长连接的相关参数，提高nginx的通信性能。
posturl: nginx-keepalive
date: 2018-09-24 17:33:15
---

### 基础知识
1. http协议是在应用层的，它的传输层使用的协议是tcp。因此发出http请求之前，需要先建立tcp连接。
2. tcp连接需要三次握手，释放连接需要四次挥手。tcp连接释放之后，主动关闭连接一端还需要维持连接的TIME_WAIT状态的2MSL的时间。
3. 通过使用keep-alive机制，对连接进行复用，减少tcp连接建立与释放，从而减少tcp握手挥手次数，也意味着可以减少TIME_WAIT状态连接，节约资源。

### 与客户端相关的三个http keepalive指令
1. `keepalive_disable`。该指令用于关闭特定浏览器的keep-alive特性。语法`keepalive_disable none |browser name`。none 为允许所有浏览器的keep-alive特性。默认值为`msie6`,ie
2. `keepalive_requests`。该指令用于指定每个连接可以完成的最大请求数，也就是复用次数，达到次数后，连接关闭。语法`keepalive_requests number;`，number为次数。默认可以复用次数为100。
3. `keepalive_timeout`。该指令用于配置keep-alive的时间，也就是保活时长。语法`keepalive_timeout timeout [header_timeout];`。第一个参数timeout为连接保活时间，默认保活时间是75秒。第二个参数header_timeout为http响应报文头中的值，设置之后，响应头中包含`Keep-Alive: timeout=header_timeout`。

以上三个指令在配置中的上下文均为`http, server, location`。另外`keepalive`指令与客户端无关。

### upstream中的keepalive指令
nginx的`keepalive`指令是应用在`upstream`，其含义不是保活时间。可以说跟http的keep-alive机制无关。`upstream`中的`keepalive`是指每个worker进程缓存中的最大的**空闲**连接数量。
查看官方文档得知，从`1.15.3`版本起，`upstream`中也支持`keepalive_requests`和`keepalive_timeout`指令，但是这个是配置与上游服务的连接保持时间以及复用次数的，与客户端无关。

另外，在`location`中加入如下设置:
```
proxy_http_version 1.1; # 指定与上游服务器使用的http协议版本1.1
proxy_set_header Connection ""; # 清除客户端带来的connection
```

### http1.0与http1.1中的keep-alive区别
http1.0默认不开启`keep-alive`, 开启`keepalive`需要在头部中包含一个`Connection: Keep-Alive`的头域。http1.1默认支持持久连接，关闭需要在头部中包含一个`Connection: close`头域。

另外，http中的保活时间等设置是建议，不是承诺。
