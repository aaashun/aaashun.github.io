---
layout: post
title: android应用的逆向
comments: true
tags: android
---
年前两月去x项目跑龙套了, 负责解决几个关键问题, 996工作, 没写文章, 也没时间运动, 回来都长肿了。基本上都是之前没涉及过的技术, 基于当时的笔记, 这几天简单分享一下.
第一个为了快速推进原型开发, 不得已对一商业sdk做了破解, 解除了授权限制, root权限, 改了部分代码, 加了日志, 改了点参数等. (声明: 目前已使用授权的SDK)

用到的一些工具:

1. <a href="http://www.android-decompiler.com">jeb</a>
 
    用于android apk的直接反编译, 浏览反编译后的java代码, 比jd-gui更准确也更易用, 但遇到可疑的反编译代码时可与jd-gui对比

2. zip

    apk其实就是zip压缩包, zip用于释放出apk中的classes.dex, assets, res等目录和文件

3. <a href="http://code.google.com/p/dex2jar">dex2jar</a>

   dex2jar是一个工具集, 主要用于classes.dex与*.class的相关操作, 例如从classes.dex里释放出.class文件, 把.class文件回打包到classes.dex. 另外d2j-apk-sign工具可用于apk文件的签名.

4. <a href="http://code.google.com/p/smali">smali/baksmali</a>

    用于将class文件反汇编成smali语法的文本文件, smali是android虚拟机的指令格式, 修改smali文件即相当于修改了对应的java代码. smali代码不会写没关系, 可以这样偷懒: 在eclipse里用java代码写好代码片段, 让开发工具自动生成apk, 然后配合zip+dex2jar+smali就能获得这段java代码对应的smali代码了.

5. <a href="https://www.hex-rays.com/products/ida">ida pro</a>

    专业的反汇编利器, 想用好的话怎么也得懂点汇编知识, 很多app的重要特性, 比如加密, 授权, 付费, 广告等的实现都是放在lib动态库里的

6. <a href="http://www.vim.org">vim</a>

    配合xxd, 修改二进制的so文件, 用windows的同学也可以使用ultraedit

工具都列完了, 修改完.class或.so文件后重新制作apk也很简单, 先用zip将classes.dex, assets, res等打包, 再用dex2jar里的d2j-apk-sign签个名就可以了


除了工具之外, 至少具备如下了些知识:

 * android应用开发
 * java, c语言, 汇编基本知识

有时候运气也很重要, 比如java代码是混淆过的, 反编译后看到的类名和变量名都是a, B, C这类无规则的命名, 只能根据逻辑来猜测, 曾经和混淆过的代码, 眼都花了, 好在jeb可以很方便地重命名. 再比如so里反汇编过的代码, 需要猜相关字符串, 函数名, 比如要绕过授权, 先搜check, license之类关键词再上下找jmp, cmp之类比较或跳转的指令, 很可能忙活好几天最后只需要改一两个字节.

网上有也不少例子可以参考的
<a href="http://www.52pojie.cn/thread-225364-1-1.html"></a>
<a href="http://www.2cto.com/kf/201412/357218.html"></a>
