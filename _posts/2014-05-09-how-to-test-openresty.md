---
layout: post
title: how to test openresty
comments: true
tags: openresty nginx test
---

5月份的时候为openresty/lua-nginx-module<a href="https://github.com/openresty/lua-nginx-module/pull/367">实现全双工cosocket</a>时写测试用例, 翻了半天资料才写完测试用例, 这篇文章其实是当时的一个笔记, 后边的几个参考文章比正文更值得看.

本文假设你学习过perl编程, 且了解perl的测试框架Test::Simple, Test::More, Test::Class等, 亦可根据结尾的参考文章快速入门.

在开发openresty/nginx模块时, 需要针对具体某一特性做单元测试, Test::Nginx就是为满足这一需求而诞生的一个测试框架(春哥出品). 目前很多nginx模块都是用Test::Nginx做的单元测试, 例如`ngx_lua`, `ngx_echo`等
如下是简单的入门使用教程.

<h2>1. 安装Test::Nginx模块</h2><br/>

先检查是否已安装Test::Nginx模块, 若已安装则跳过这一步
<pre>
perl -e 'use Test::Nginx'
</pre><br/>

安装perl的模块管理程序<a href="http://www.cpan.org/">CPAN</a>, 类似apt, yum
<pre>
sudo yum install perl-CPAN
sudo cpan> o conf init
</pre><br/>

安装YAML(若已安装则跳过)
<pre>
sudo cpan> install YAML
</pre><br/>

安装Text::Nginx模块
<pre>
sudo cpan> install Test::Nginx
</pre><br/>


<h2>2. 第一个测试用例</h2><br/>

保存如下测试代码到文件t/foo.t
{% highlight perl %}
use Test::Nginx::Socket;

repeat_each(1);
plan tests => 2 * repeat_each() * blocks();

$ENV{PATH} .= ":/usr/local/openresty/nginx/sbin";  # make sure 'nginx' is in $PATH

run_tests();

__DATA__

=== TEST 1: sanity
--- config
location /foo {
    content_by_lua 'ngx.print("hello, world");';
}
--- request
GET /foo HTTP/1.0
--- response_body_like: ^hello, world$
--- error_code: 200
{% endhighlight %}

执行单个测试用例
<pre>
perl t/foo.t
</pre><br/>

测试通过返回
<pre>
1..2
ok 1 - TEST 1: sanity - status code ok
ok 2 - TEST 1: sanity - response\_body\_like - response is expected (hello, world)
</pre><br/>

----
参考:

* <http://jianlee.ylinux.org/Computer/Perl/perl_base.html> - Perl基本语法
* <http://www.ibm.com/developerworks/cn/opensource/os-cn-perl-unittesting/index.html> - Perl 脚本中单元测试自动化浅析
* <http://agentzh.org/misc/slides/BJPW2009/perl-testing.html> - agentzh's slide
* <http://chenxiaoyu.org/2013/12/04/test-nginx.html> - Test::Nginx模块介绍
* <http://www.taobaotest.com/blogs/show/2433> - Test::Nginx 测试框架分析
* <http://en.wikipedia.org/wiki/Test_Anything_Protocol> - TAP
* <http://testanything.org> - TAP
* <https://github.com/openresty/test-nginx> - Test::Nginx github
