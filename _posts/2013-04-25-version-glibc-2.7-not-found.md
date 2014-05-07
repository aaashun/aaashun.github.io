---
layout: post
title: version `GLIBC_2.7′ not found, 让程序兼容旧系统
tags: 
---

最近一个客户要在centos5.4上调用在centos6.0上编译的库文件, 报错version `GLIBC_2.7′ not found, 尝试解决过程如下

<h2>1. 查看glibc版本</h2><br/>

<pre>
#ldd –version
ldd (GNU libc) 2.12
</pre>

centos 5.4用的是glibc2.4, centos6.0用的是glibc2.12 (注gcc的版本号和glibc是对应的)

<h2>2. 查看可执行文件版本依赖信息</h2><br/>

<pre>
#readelf -V {BINARY}
Version symbols section ‘.gnu.version’ contains 98 entries:
Addr: 00000000004010bc  Offset: 0x0010bc  Link: 5 (.dynsym)
000:   0 (*local*)       2 (GLIBC_2.2.5)   3 (GLIBC_2.2.5)   2 (GLIBC_2.2.5)
004:   3 (GLIBC_2.2.5)   3 (GLIBC_2.2.5)   4 (GLIBC_2.2.5)   2 (GLIBC_2.2.5)
008:   3 (GLIBC_2.2.5)   3 (GLIBC_2.2.5)   0 (*local*)       0 (*local*)
00c:   3 (GLIBC_2.2.5)   3 (GLIBC_2.2.5)   5 (GLIBC_2.7)     3 (GLIBC_2.2.5)
</pre><br/>

可以看到依赖库的版本信息, 这里能看到GLIBC_2.7
<br/>
<br/>
<h2>3. 查看哪个函数依赖glibc2.7</h2><br/>

<pre>
#objdump -t {BINARY} | grep GLIBC_2.7
0000000000000000       F *UND*  0000000000000000              __isoc99_sscanf@@GLIBC_2.7
</pre>

或

<pre>
\#nm -A {BINARY} | grep GLIBC_2.7
{BINARY}:                 U __isoc99_sscanf@@GLIBC_2.7
</pre><br/>

确定只有__isoc99_sscanf这个函数依赖GLIBC_2.7, 可以在/usr/include/stdio.h里看类似如下的宏定义

{% highlight c %}
#if defined __USE_ISOC99 && !defined __USE_GNU
#  define fscanf __isoc99_fscanf
#  define scanf __isoc99_scanf
#  define sscanf __isoc99_sscanf
# endif
#endif
{% endhighlight %}

由上可知可以通过修改编译参数-std=c89或-ansi或-D _GNU_SOURCE来规避__isoc99_sscanf

由于不同版本系统的gcc和glibc版本不同, 最好的方法是在指定版本的系统上完成编译.

----
参考:

* http://www.evanjones.ca/portable-linux-binaries.htm
