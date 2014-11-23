---
layout: post
title: 运营商在80端口的劫持
comments: true
tags: http pipeline keepalive hijack
---

先说一下结论: <b>由于运营商在80端口的劫持行为, 请谨慎在80端口的http web服务上使用http1.1的pipeline, keepalive, 或websocket协议</b>

我们所使用的协议都是基于http/1.1的, 不管是pipeline, keepalive还是websocket, 都是严格遵守标准协议的, 下边我们就来见识一下各种劫持现象

## 1. 各种劫持现象 
### 1.1 长连接上的pipeline requests被改成多个短连接request

问题网关: 北京电信3G 106.37.8.231

<img src="/img/http-pipelining.png"/><br/>
图1. non-pipelined vs. pipelined connection

使用pipeline技术主要是为了以最快速度将语音传输到服务端, 避免等待上一个请求的响应而延迟发送, 客户端和服务端都是我们自已实现的, 支持http pipeline是没有问题的.

但是在出问题时我们抓到的报文如下:

<img width=800 height=200 src="/img/http-pipelining-client-side-sniffer.png"/><br/>
图2. pipelining客户端抓包<br/>

<img width=800 height=300 src="/img/http-pipelining-server-side-sniffer.png"/><br/>
图3. pipelining服务端抓包<br/>

奇怪的现象, 在客户端看来与服务端维护的是一条长连接, 但是从服务端看到的是多个连接, 即每个请求使用不同的连接.

### 1.2 长连接上的请求被丢弃

问题网关: 北京移动4G 221.179.140.171

在长连接上的请求序列是:

    1. POST /api/v3.0/score (text/plain)
    2. POST /api/v3.0/score (application/oct-stream)
    3. POST /api/v3.0/score (application/oct-stream)
    ...
    n. POST /api/v3.0/score (text/plain)

下图你没看错, 服务端抓包所看到的请求里content-type为application/oct-stream的请求没了

<img width=800 heigth=400 src="/img/http-keepalive-discard.png"/><br/>
图4. 请求丢失<br/>

### 1.3 修改header

问题网关: 上海电信3G 101.89.195.219

比如典型的websocket握手请求的header

    GET / HTTP/1.1
    Host: server.example.com
    Upgrade: websocket
    Connection: Upgrade
    Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
    Sec-WebSocket-Version: 13

但是我们却经常收到下图这种请求头, Upgrade被删除, Connection的值被改成了keep-alive, 增加了Cache-Control

<img width=800 heigth=400 src="/img/http-header-changed.png"/><br/>
图5. http header修改


## 2. 解决

一开始我们觉得作为运营商不管从技术规范还是道德上都不应该对应用层的数据做作何修改, 事后想想还是自已too young too simple. 我们一直在想这个网关/防火墙的行为是怎样的, 怎样绕过去, 走了不少弯路, 尝试了各种解决方案, 最后我们换一个非80的端口, 发现问题神奇地消失了. 之前之所以使用80端口是因为有个别防火墙会禁止非80端口的通信.

最后的解决方案是优先使用非80端口, 如果不能访问再切换到80端口, 尽量绕开该死的80端口劫持问题.(注: 因SSL近4KB的握手数据在移动网络下传输会影响用户体验, 暂未使用)

建议: 

* <b> 如果你是web服务提供者, 从安全考虑, 你应该尽量使用https, 像google那样负责 </b>
* <b> 如果你是普通用户, 从安全及隐私考虑, 你应该尽量使用<a href="http://ugetvpn.com/?r=e87ae4edb294892a">vpn</a>, 你会发现不一样的世界, 你懂的 </b>


----
参考:

* http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.42
* https://en.wikipedia.org/wiki/HTTP/1.1_Upgrade_header 
* http://tools.ietf.org/html/rfc7230#section-6.3.2, https://en.wikipedia.org/wiki/HTTP_pipelining
* https://en.wikipedia.org/wiki/Chunked_transfer_encoding
* http://theiphonewiki.com/wiki/Siri_Protocol
