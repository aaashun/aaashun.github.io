---
layout: post
title: 运营商在80端口的劫持
comments: true
tags: 3G 4G http pipeline keepalive
---

这周收到客户反馈在北京和上海某些地区通过3G或4G上网卡无法使用我们提供的语音识别服务, 最终我们是解决了这个问题.

先说一下结论: <b>由于运营商在80端口的劫持行为, 请谨慎在80端口的http web服务上使用http1.1的pipeline, keepalive, 或websocket协议</b>

我们原先所使用的协议是基于http/1.1的, 在一个http长连接上通过多个pipeline的request实时将用户录音发送至识别服务器, 这样识别服务器便可以在用户录音时就开始复杂的解码运算.

<img src="/img/http-pipelining.png"/><br/>
图1. non-pipelined vs. pipelined connection

使用pipeline技术主要是为了以最快速度将语音传输到服务端, 避免等待上一个请求的响应而延迟发送, 客户端和服务端都是我们自已实现的, 支持http pipeline是没有问题的. 但是在出问题时我们抓到的报文如下:

<img src="/img/client-side-sniffer.png"/><br/>
图2. 客户端抓包<br/>

<img src="/img/server-side-sniffer.png"/><br/>
图3. 服务端抓包<br/>

看到奇怪的现象了吧, 在客户端看来与服务端维护的是一条长连接, 但是从服务端看到的是多个连接, 即每个请求使用不同的连接.

mobile client -> IP gateway of ISP -> firewall -> server <br/>
图4. 逻辑架构 <br/>

从图4可以看出, 在移动客户端要访问服务端需经过运营商的IP网关, 我们分析应该是运营商的IP网关分析了应用层HTTP协议, 自作聪明地改变传输层的连接. 于是我们想了一系列方案来尝试绕过运营商的挟持.

方案1: 在建立连接时通过应用层的握手来完成协议升级(http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.42, https://en.wikipedia.org/wiki/HTTP/1.1_Upgrade_header)

握手请求如下:
    GET /service HTTP/1.1
    Host: example.com
    Upgrade: myprotocol
    Connection: Upgrade

握手响应如下:
    HTTP/1.1 101 Switching Protocols
    Upgrade: myprotocol
    Connection: Upgrade

握手成功后相当于使用自定义的协议

这招还真有效, 当场解决了北京一个电信3G用户的使用问题, 从服务端抓包看一切正常, 多个请求走的是一个连接, 只是请求的header里被插入几个运营商的header.

但是第二天我们就发现了一上海电信3G用户还是不能用, 我们从服务端抓包看到的是请求header变成了:
    GET /api/v3.0/score HTTP/1.1
    Host: s.api.aispeech.com
    Cache-Control: max-age=43200
    Connection: keep-alive

header里的Upgrade被删除了, Connection的取值也被改成了keep-alive, Cache-Control是运营商插入的, 我们只能尝试第二种方案了.

方案2: websocket

一开始我们觉得websocket是公开的协议, 应该不会连这个都不支持吧, 测试下来证明还是我们too young too simple. 测试很简单, 使用http://www.websocket.org/echo.html上的EchoTest示例, 连接自已的websocket server, 发现在出问题的情况下Upgrade和Connection两个header被删除或修改.

为了验证是运营商的IP网关修改了协议, 让mobile client连接北京, 上海, 杭州等地的不同出口线路的机房均收到同样的错误请求报文, 于是我们很努力地

方案3: 

1. 在北京发现的问题

client与server之间建立的是一条tcp连接, 在tcp连接之上的应用层协议是http 1.1 (参考http://tools.ietf.org/html/rfc7230#section-6.3.2, https://en.wikipedia.org/wiki/HTTP_pipelining)
但是从服务端抓包看到的即是多个tcp连接, 每个连接只处理一个http请求

我们目前分析的结论是电信3G网关在做NAT时根据应用层协议内容, 修改了传输层的tcp连接, 破坏了协议的完整性


2. 在上海发现的问题

为修复北京发现的问题, 我们在建立连接时通过应用层的握手来完成协议升级(参考http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.42, https://en.wikipedia.org/wiki/HTTP/1.1_Upgrade_header)

握手请求如下:
    GET /api/v3.0/score HTTP/1.1
    Host: s.api.aispeech.com
    Upgrade: httpas
    Connection: Upgrade

握手响应如下:
    HTTP/1.1 101 Switching Protocols
    Upgrade: httpas
    Connection: Upgrade

握手成功后相当于使用自定义的协议

但是从服务端抓包看到的握手报文被修改成如下:
    GET /api/v3.0/score HTTP/1.1
    Host: s.api.aispeech.com
    Cache-Control: max-age=43200
    Connection: keep-alive

其中header里的Upgrade和Connection两个字段被删除或修改


