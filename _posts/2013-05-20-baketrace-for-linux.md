---
layout: post
title: 程序debug之backtrace - linux
tags: debug
---

backtrace即回溯堆栈, 其运行原理是分当前堆栈, 定位上层函数在栈中的帧地址, 再分析上层函数的堆栈，定位更上层函数在栈中的帧地址, 依此类推.

我经常用此定位异常,可用于查看当前函数的调用关系.

这里我将介绍其基本使用方法, 以及程序异常时自动打印堆栈.

<h2> 1. 基本使用 </h2><br/>

示例代码如下

{% highlight c %}
/* backtrace_foo1.c */
#include <execinfo.h>
#include <stdio.h>
#include <stdlib.h>

#define BACKTRACE() \
do {\
    void *array[20];\
    size_t size;\
    char **strings;\
    size_t i;\
    size = backtrace(array, 20);\
    strings = backtrace_symbols(array, size);\
    for (i = 0; i < size; i++) {\
        printf ("%s\n", strings[i]);\
    }\
    free (strings);\
} while(0)

void func1()
{
    BACKTRACE();
}

void func()
{
    func1();
}

int main(int argc, char **argv)
{
    func();
    return 0;
}
{% endhighlight %}<br/>

<pre>
gcc backtrace_foo1.c
./a.out
</pre><br/>

运行结果如下:
<pre>
./a.out() [0x4005c3]
./a.out() [0x400630]
./a.out() [0x40064b]
/lib64/libc.so.6(__libc_start_main+0xfd) [0x35be01ecdd]
./a.out() [0x4004e9]
</pre><br/>

<h2>2. 使用addr2line显示函数名和行号</h2><br/>

上边只显示了地址没有具体函数名, 没关系, 可以用addr2line查看函数名和行号
<pre>
echo 0x4005d2 | addr2line -e a.out -f
</pre><br/>

错误显示如下
<pre>
func1
bracktrace_foo1.c:0
</pre><br/>

注意行号显示不对, 在编译时加上-g参数就行了
<pre>
gcc -g backtrace_foo1.c
./a.out
echo 0x4005d2 | addr2line -e a.out -f
</pre><br/>

正确显示如下
<pre>
func
backtrace_foo1.c:21
</pre>

<h2>3. 直接打印函数名</h2><br/>

根据man backtrace, 在编译时加上-rdynamic参数可以直接打印函数名
<pre>
gcc -rdynamic backtrace_foo1.c
./a.out
</pre>

显示如下
<pre>
./a.out(func1+0x1f) [0x400803]
./a.out(func+0xe) [0x400870]
./a.out(main+0x19) [0x40088b]
/lib64/libc.so.6(__libc_start_main+0xfd) [0x35be01ecdd]
./a.out() [0x400729]
</pre>

这种方法有一个问题是static函数名不会被打印出来, 所以还是用addr2line吧

<h2>4. 在程序崩溃时自动打印堆栈</h2><br/>

实现思路是在程序启时注册SIGSEGV, SIGFPE, SIGBUS等异常信号的处理函数, 这些异常信号的处理函数是由库函数回调的, 所以在处理函数里打印的堆栈信息里很容易找到崩溃点

示例代码2
{% highlight c %}
/* backtrace_foo2.c */
#include <execinfo.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

#define BACKTRACE() \
do {\
    void *array[20];\
    size_t size;\
    char **strings;\
    size_t i;\
    size = backtrace(array, 20);\
    strings = backtrace_symbols(array, size);\
    for (i = 0; i < size; i++) {\
        printf ("%s\n", strings[i]);\
    }\
    free (strings);\
} while(0)

void func1()
{
    char *p = NULL;
    printf("%s\n", p);
}

void func()
{
    func1();
}

static void signal_handler(int sig)
{
    switch(sig)
    {
        case SIGSEGV: /* segmentation fault */
        case SIGFPE:  /* erroneous arithmetic operation */
        case SIGBUS:  /* bus error */
            BACKTRACE();
            exit(EXIT_FAILURE);
            break;
        default:
            break;
    }
}

int main(int argc, char **argv)
{
    signal(SIGSEGV, signal_handler);
    signal(SIGFPE,  signal_handler);
    signal(SIGBUS,  signal_handler);

    func();
    return 0;
}
{% endhighlight %}<br/>

编译执行
<pre>
gcc -rdynamic backtrace_foo2.c
./a.out
</pre><br/>

打印如下, 第1行是异常信号处理函数, 第2行到第4行是库函数, 第五行即是异常点
<pre>
./a.out() [0x4008f7]
/lib64/libc.so.6() [0x35be032900]
/lib64/libc.so.6() [0x35be07fe21]
/lib64/libc.so.6(_IO_puts+0x1b) [0x35be067f2b]
./a.out(func1+0x1c) [0x400890]
./a.out(func+0xe) [0x4008a0]
./a.out(main+0x46) [0x4009ae]
/lib64/libc.so.6(__libc_start_main+0xfd) [0x35be01ecdd]
</pre><br/>
