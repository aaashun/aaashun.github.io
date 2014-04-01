---
layout: post
title: mingw-w64安装及与aimake的集成
---

今天用了一下mingw-w64(http://mingw-w64.sourceforge.net/), 用下来感觉比mingw兼容性更强, 比如pthread_t是整型而不是奇葩的结构体.
mingw只能用来编译32位, mingw-w64可以编译32位和64位程序, 一个伟大的开源项目.

<h2> 安装mingw-get </h2>

下载安装[mingw-get-setup.exe] [1], 安装目录选E:/MinGW64, 其它啥也别选, 结束退出
  [1]: http://www.mingw.org/wiki/Getting_Started

<h2> 安装msys和常用工具 </h2>

    c:/mingw/bin/mingw-get.exe install msys-coreutils msys-vim msys-zip msys-unzip msys-sed msys-gawk

安装好后就有一个类linux终端的MinGWShell可以使用了, 可以为脚本E:/MinGW64/msys/1.0/msys.bat创建一个快捷方式

<h2> 安装制作dll所需的工具lib和link </h2>
从visual studio安装目录的VC/bin中找到lib.exe, link.exe, pgodb100.dll, msdis170.dll, mspdb100.dll

    mkdir -p /mingw/vc/bin

把上边的exe和dll复制到/mingw/vc/bin目录下, 并修改环境变量

    echo "export PATH=\$PATH:/mingw/vc/bin" >> /etc/profile

<h2> 安装gcc toolchains </h2>

这些都是mingw-w64项目下的工具链, 推荐rubenvb编译的版本, 命名格式是: [x86_64-w64-mingw32(目标平台)]-[gcc-4.8.0]-[win64(当前系统)]_rubenvb

* x86_64(64位): [x86_64-w64-mingw32-gcc-4.8.0-win64_rubenvb.7z] [1]
* i686(32位):  [i686-w64-mingw32-gcc-4.8.0-win64_rubenvb.7z] [2]
  [1]: http://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win64/Personal%20Builds/rubenvb/gcc-4.8-release/rubenvb/x86_64-w64-mingw32-gcc-4.8.0-win64_rubenvb.7z/download
  [2]: http://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win32/Personal%20Builds/rubenvb/gcc-4.8-release/i686-w64-mingw32-gcc-4.8.0-win64_rubenvb.7z/download

下载好后解压到c:/mingw/toolchains目录下

<h2> 编译示例程序 </h2>

程序内容

    /* foo.c */
    #include <stdio.h>
    int main(int argc, char **argv)
    {
        printf("%d\n", sizeof(void *));
        return 0;
    }

编译成32位, 执行结果应该是4

    /c/MinGW/toolchains/i686-w64-mingw32-gcc-4.8.0-win64_rubenvb/mingw32/bin/i686-w64-mingw32-gcc foo.c

编译成64位, 执行结果应该是8

    /c/MinGW/toolchains/x86_64-w64-mingw32-gcc-4.8.0-win64_rubenvb/mingw64/bin/gcc foo.c

<h2> 使用aimake </h2>

aimake是基于gnu make的整合交叉编译工具, 目前支持mingw, android, ios, linux等目标系统平台的编译

项目网址: https://github.com/ashun/aimake

下载安装好后注意配置一下aimake/mingw/init.mk下边的路径变量

使用aimake编译foo.c

编写aimakefile

    /* aimakefile */
    LOCAL_MODULE = foo
    LOCAL_SRC_FILES = foo.c
    include $(BUILD_EXECUTABLE)

编译32位

    aimake clean all 或 aimake MINGWABI=i686 clean all

编译64位

    aimake MINGWABI=x86_64 clean all
