---
layout: post
title: 做移动开发应该了解的arm及编译优化
comments: true
tags: arm aimake
---

这两天在做SDK兼容性测试时遇到了不少问题, sdk要兼容高中低端的android手机, 这个问题之前遇到过, 也查过一些资料, 今天整理一下.

<h2> 1. ARM处理器系列 </h2>

<h3> arm7系列 </h3>

1994年推出, 世界上使用范围最广的32位嵌入式处理器, 年代太久远

<h3> arm9系列 </h3>

http://www.arm.com/zh/products/processors/classic/arm9/index.php

* Hz: 200MHz - 400MHz
* arch: armv5te
* isa: arm, thumb
* fpu: vfpv2(optional)

1997年推出, 大学时用过, 现在的电信送的免费android手机很多是用这个处理器, 代表机型: 诺基亚N73, 华为C8500

<h3> arm11系列 </h3>

http://www.arm.com/zh/products/processors/classic/arm11/index.php

* Hz: 800MHz - 1GHz
* arch: armv6
* isa: arm, thumb
* fpu: vfpv2(optional)

2003年问世, 现在500元以内手机基本都是这个系列, 代表机型: HTC G8, 三星s5830

<h3> cortex-a系列 </h3>

这个系列是目前的主流, 所以这里讲细点

<h4> cortex-a5 </h4>

* Hz: 400MHz - 800MHz
* arch: armv7a
* isa: arm, thumb, thumb-2
* fpu: vfpv4(optional), neon(optional)

定位于低端千元机, 代表机型: HTC Desire V, 华为U8825

<h4> cortex-a8 </h4>

* Hz: 600MHz - 1GHz
* arch: armv7a
* isa: arm, thumb, thumb-2
* fpu: vfpv3, neon

单核解决方案, 代表机型: iphone4, 三星I9000, 摩托罗拉里程碑2

<h4> cortex-a9 </h4>

* Hz: 800MHz - 2GHz
* arch: armv7a
* isa: arm, thumb, thumb-2
* fpu: vfpv3(optional), neon(optional)

多核解决方案, 目前市场上2核/4核的多数是这个, 代表机型: iphone5, 三星I9000

<h4> cortex-a7 </h4>

* Hz: 800MHz - 1.2GHz
* arch: armv7a
* isa: arm, thumb, thumb-2
* fpu: vfpv4, neon

低功耗, 市场上较少, 定位于低端千元机市场, 代表机型: 联发科MT6589

<h4> cortex-a12 </h4>

* Hz: ?
* arch: armv7a
* isa: arm, thumb, thumb-2
* fpu: vfpv4(?), neon(?)

将于2014年上市, 定位于中端市场, 比cortex-a9性能提升40%, 尺寸小30%, 将取代cortex-a9

<h4> cortex-a15 </h4>

* Hz: 1GHz - 2.5GHz
* arch: armv7a
* isa: arm, thumb, thumb-2
* fpu: vfpv4, neon

目前性能最高, 1-8核, 主要应用在一些平板电脑上和高端机皇上, 代表机型: 三星GALAXY S4


<h2> 2. ARM处理器架构(arch) </h2>

http://www.arm.com/zh/products/processors/instruction-set-architectures/index.php

<h3> armv5te </h3>

arm9系列使用该架构

<h3> armv6 </h3>

arm11系列使用该架构

<h3> armv7-a </h3>

cortex-a系列使用该架构

<h3> armv8 </h3>

最新架构, 应该还没应用

<h2> 3. ARM指令集(isa) </h2>

<h3> thumb </h3>

16位的指令集, 每个thumb指令都对应一个32位的arm指令, 它仅是将32位arm指令的压缩成16位的指令编码方式, 目标是低功耗, 代码小.

如下是和arm指令集的对比:

* Thumb代码所需的存储空间约为ARM代码的60％～70％。
* Thumb代码使用的指令数比ARM代码多约30％～40％。
* 若使用32位的存储器，ARM代码比Thumb代码快约40％。
* 若使用16位的存储器，Thumb代码比ARM代码快约40％～50％。
* 与ARM代码相比较，使用Thumb代码，存储器的功耗会降低约30％

由此可见若对性能要求高应使用32位的存储系统和arm指令集, 若对要求低功耗应使用16位的存储器和thumb指令集

<h3> thumb-2 </h3>

16位/32位的指令集, 是对thumb指令集的扩充, 增加了一些32位指令, 改善thumb指令集的性能

<h3> arm </h3>

32位的指令集, 可以实现arm架构下所有功能

<h2> 4. ARM浮点运算单元(fpu) </h2>

<h3> vfpv1 </h3>

已废弃 

<h3> vfpv2 </h3>

armv5te, armv6架构中的浮点计算指令集

<h3> vfpv3 </h3>

部分armv7a架构中的浮点计算指令集

<h3> vfpv4 </h3>

部分armv7a架构中的浮点计算指令集

<h3> neon </h3>

多媒体处理单元, 提供128位的SIMD指令集, 类似x86上的sse, 应用于cortex-a系列处理器上, 性能最好

<h2> 5. GCC相关编译参数 </h2>

<h3> -march </h3>

armv5te, armv6, armv7-a

<h3> -mfloat-abi </h3>

soft, softfp, hard. soft是软浮点, 效率最低, 适合于没有fpu的ARM处理器; softfp是默认设置, 由fpu完成浮点计算, 但函数参数的传递使用通用的整型寄存器而不是FPU寄存器; hard效率最高, 由fpu完成浮点计算, 且使用fpu浮点寄存器将函数参数传递给fpu处理.

<h3> -fpu </h3>

vfp, vfpv3, vfpv4, neon. 需要配合float-abi及其它参数一起使用, 例如: -ftree-vectorize -mfpu=neon -mfloat-abi=softfp

<h3> -marm </h3>

使用arm指令集

<h4> -mthumb </h4>

使用thumb指令集, 编译器会根据-march自动使用thumb-2的指令集

----
看吧, 想要提供一个兼容各android手机的底层库不是那么简单的, 如果再想针对性优化一下, 至少可以有如下几种组合:

* armv5te
* armv5te-vfpv2
* armv6
* armv6-vfpv2
* armv7a
* armv7a-vfpv3
* armv7a-vfpv4
* armv7a-neon

PS: Android NDK/Xcode提供的那几个参数是不方便做如上针对性优化的, 建议使用aimake, 调整aimake/android/init.mk里的CFLAGS就可以了, https://github.com/ashun/aimake

----
参考:

* ARM处理器 -- http://www.arm.com/zh/products/processors/index.php
* ARM浮点运算单元 -- http://www.arm.com/zh/products/processors/technologies/vector-floating-point.php
* ARM架构 -- http://baike.baidu.com/view/4078025.htm
* thumb指令集 -- http://blog.csdn.net/liuchao1986105/article/details/6544530
* GCC使用VFP -- http://doc.ironwoodlabs.com/arm-arm-none-eabi/html/getting-started/sec-armfloat.html
* vfp--arm浮点体系结构机介绍 -- http://www.cnblogs.com/Akagi201/archive/2012/03/31/2427063.html
* ARM内核全解析，从ARM7到Cortex-A9 -- http://www.myir-tech.com/resource/448.asp
* 从ARM9到A15 手机处理器架构进化历程 -- http://tech.163.com/mobile/12/0413/06/7UV0ESSO00112K8D_all.html
