---
layout: post
title: 程序debug之backtrace - windows
comments: true
tags: debug
---

我们的程序在windows上编译都是用mingw编的, 偶尔也会遇到崩溃的问题, 定位时当然需要打印堆栈信息. 查询了一下发现云风已经完成了<a href="https://code.google.com/p/backtrace-mingw/">一个版本</a>可以直接使用

参考:

* http://blog.codingnow.com/2010/07/mingw_stack_backtrace.html
* http://stackoverflow.com/questions/5693192/win32-backtrace-from-c-code
