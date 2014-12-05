---
layout: post
title: openresty用于计算密集型服务
comments: true
tags: nginx lua openresty
---

这篇文章更多的是记录了设计的过程, 不仅仅是设计的结果.

### 1. 原服务那些事
时间回到2011年, 几个初生牛犊的工程师用c语言完整实现了server/client, server端又是分三层: httpa, core-proxy, core-service. server端实现时主要参考了nginx, 但是仅实现了我们所需要的特性, 当时的两个NB理由:

* 不使用第三方的代码, 我们要自已完全可控
* 我们能实现得更好

<center>
<img width=400 src="/img/old-arch.png"/><br/>
图1. 原服务端架构图
</center>

时过境迁, 2年后当年NB服务的诸多问题了慢慢呈现出来:

* 代码结构设计不合理
* 代码风格奇特, 可读性差
* 文档不全, 新人无从快速上手
* 很难扩展新功能
* 性能差, httpa/core-proxy仅支持1k并发
* 整体架构层次太多, 部署维护很费神

到后来已经渐渐没人愿意, 也没人有能力维护好这一代的服务程序了, 再也跟不上业务发展的需要了.

### 2. 基于openresty的新服务
2014年初我们开始重新思考计算密集型服务程序的设计, 只有一个核心目标: 简单, 一定要简单. 简单才易于开发和维护, 简单才易于整体架构, 简单才易于扩展, 简单才易于做到稳定, 只有做简单才有资格谈性能, 这也是UNIX哲学里最重要的KISS原则.

这次从开始就没打算自已撸, 由于略懂nginx, 平时又受益于lua, 便尝试openresty(nginx + ngx_lua)作为计算服务的容器.

原本nginx作为单线程EventLoop的代表, 以其每秒轻松5万的QPS的出色性能, 一般是作为网络IO密集型的web服务器或代理服务器. openresty的出现让nginx具备应用服务器的能力, 但作为计算服务容器还有两个主要问题需要解决:

* 计算与主循环分离
* 与核心c语言计算代码集成

openresty的lua代码执行是在事件循环里的, 在lua代码里不能有任何复杂的计算或其它可能阻塞主循环的代码, 常见的做法是通过fastcgi与计算服务通信, 就像php-fpm那样 

<center>
<img width=380 src="/img/nginx-fastcgi.png"/> <br/>
图2. nginx + fastcgi
</center>

fastcgi在这里只是代理的角色, 让应用服务程序与web服务程序解耦, 这样应用服务程序便可以用python, php等其它开发语言实现了. fastcgi的确可以解决这两个问题, 但是想利用lua作为胶水语言还得费点事. 所以我们又尝试完全在openresty里做, 如下图所示:

<center>
<img width=450 src="/img/all-in-openresty.png" /> <br/>
图3. all in openresty
</center>

#### 2.1 只使用一个nginx worker
因为是计算密集型服务, 一台服务器只能处理不到一百个并发请求甚至更少, 一个nginx worker处理网络IO足够了.

#### 2.2 posix.fork()创建计算进程 
ngx_lua没有提供fork方法, 我使用的是luaposix, luaposix提供了丰富的posix方法的lua绑定.

#### 2.3 nginx worker和计算进程间通过tcp连接通信
为了避免block主循环, nginx worker端的服务代码里使用cosocket与计算进程通信即可.

#### 2.4 所有服务逻辑都用lua实现
lua作为服务端脚本有诸多优点:

* 设计精巧, 核心代码量只有1w多行, 编译后完整的VM小于100KB
* 内存开销小, 唯一字符串只有16B, 闭包只有24B
* 运行效率高, 加上LuaJIT更是接近c/c++
* 原生支持coroutine, 依靠coroutine完成中断/重入的切换开销很小

我倾向于所有服务逻辑都用lua编写, 一些核心的计算内核的c代码binding到lua, 作为lua模块在lua里调用.

#### 2.5 对计算进程的一些特殊要求
不能使用ngx.socket, ngx.sleep, ngx.timer, ngx.log, ngx.thread等ngx_lua里的'同步非阻塞'的方法, 在计算进程用luasocket与nginx worker进行同步阻塞的tcp通信, 在计算进程里只需埋头计算, 计算完成后直接退出进程即可.

#### 2.6 简单的进程间通信协议

<center>
<img width=300 src="/img/rpc.png" /> <br/>
图4. 简单的进程间通信协议<br/>
</center>

这只是简单的文本协议示例, 第一行是请求类型标识, 第二行是数据长度, 紧接着是数据内容. 使用文本协议, 简单直白可读性高.

#### 2.7 一些优化的设计和实现
* 共享内存<br/>
linux在fork时对内存的管理是写时拷贝(copy-on-write), 当有大量数据属于只读型时, 可以在`init_worker_by_lua`里一次性将资源加载到内存里做共享, 可以节省大量物理内存, 以及节省加载时间.

* 进程池<br/>
在我的intel i5机器上posix.fork()的耗时是毫秒级, 这当然会block住主循环的, 可以加一个进程池管理机制, 保留一些空闲进程, 同时增加计算进程的重复利用来做一定程度上的优化.

* 过载保护<br/>
增加这个机制是防止负载过高时假死, 一般有两个控制参数max\_utilization和pending\_time, max\_utilization是cpu最高使用率的阈值一般设为0.8, pending\_time是忙时等待时间一般设为2000ms. 使用率超过0.8就代表机器繁忙, 这时新的请求最多等待2000ms, 如果2000ms时间内cpu使用率降下来则处理计算请求, 否则响应server busy.

* 微服务<br/>
微服务是个架构上的概念, 在设计和实现单个计算服务时也得按微服务的思想来做. 我们把整体服务做了切分:

 * 一种类型的计算作为一个单独的服务
 * 一个nginx实例只运行一种类型的计算服务
   这样做一方面避免加载一些无用的计算资源, 另一方面每种计算任务的并发请求量相差很多, 每个计算服务需要多少配额, 运行在哪些机器上, 这些由体运维同学根据实际决定即可.
 * 不同服务用uri区分
   这样可以方便用成熟的代理程序来实现负载分发, 例如haproxy或nginx, 最终服务端逻辑架构可以简化成这样:
<center>
<img width=350 src="/img/micro-service.png"/> <br/>
图5. 新服务端架构
</center>

写在最后: 最终那5万多行c代码被简化成了几千行lua代码, 逻辑架构也是如上图那么简洁.
