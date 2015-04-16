---
layout: post
title: mtk平台lcm工作日志
comments: true
tags: android mtk lcm
---
在x项目里帮忙解决mtk平台dlp的显示问题, 之前没接解过lcm这块的驱动, 也是费了不少心思的, 理解仍为粗浅, 这里只是一些参考的汇总, 见笑. (注: 一般电路板和驱动的工作让方案商完成即可)

### android bootloader过程
参考 <a href=http://blog.csdn.net/LoongEmbedded/article/details/38269859>MTK6577+Android启动----U-Boot</a>

得知道lcm驱动会同时编进bootloader和kernel里, 显示的初始化是在bootloader里完成的, 只有在待机恢复时才会调用到kernel里的lcm get parameters接口

### lcm驱动过程
参考 <a href=http://blog.csdn.net/sunweizhong1024/article/details/8447915>LCD 驱动过程详解</a>

### lcm移植示例
参考 <a href=http://blog.csdn.net/cbk861110/article/details/17438835>MTK Android Driver: lcm</a>
代码一般位于mediatek/custom/common/kernel/lcm/xxx/yyy.c, 已有源码里可以找到不少参考的驱动, 一般只需要修改WIDTH, HEIGHT及其它显示参数即可

### lcm基础知识
参考 <a href=http://blog.csdn.net/xubin341719/article/details/9177085>AndroidLCD</a>

如下一些基本概念:

 * VBP(vertical back porch): 垂直回扫, 表示一帧图像开始时, 垂直同步信号以后无效的行数
 * VFP(vertical front porch): 垂直前扫, 表示一帧图像结束后, 垂直同步信号以前无效的行数
 * HBP(horizontal back porch): 水平回扫, 水平同步信号开始到一行的有效数据开始之间的VCLK的个数
 * HFP(horizontal front porch): 水平前扫, 一行的有效数据结束到下一个水平同步信号开始之间的VCLK的个数
 * HSYNC: HBP + HFP + Display period + pluse width; 即垂直扫描的时钟频率 = 垂直前扫 + 垂直回扫 + LCD屏的显示宽度 + 脉冲宽度; 行同步信号，表示一行数据的开始，LCD控制器在整个水平线（整行）数据移入LCD驱动器后，插入一个LCD_HSYNC信号
 * VSYNC: VBF + HBP + Display period + pluse width; 即水平扫描的时钟频率 = 水平前扫 + 水平回扫 + LCD屏的显示高度 + 脉冲宽度; 帧同步信号，表示一帧数据的开始，LCD控制器在一个完整帧显示完成后立即插入一个LCD_VSYNC信号，开始新一帧的显示；VSYNC信号出现的频率表示一秒钟内能显示多少帧图像，称为“显示器的频率”
 * VCLK: 像素时钟信号, 表示正在传输一个像素的信号
 * VDEN: 数据使能信号
 * VD: 像素数据输出端口

### 调试经验
参考 <a href=http://blog.csdn.net/bigman_123/article/details/19070579>驱动之LCD</a>, <a href=http://wenku.baidu.com/view/e2254495700abb68a982fb94.html>mtk平台LCD驱动调试</a>

示波器可以很方便确认HSYNC, VSYNC, VCLK, VDEN等信号输出是否正常


### 常用编译和刷机

LK, 内核, 框架模块, app都是可以单独编译的, 完整编译则需要要1个小时, 如下make_project.sh是对makeMtk的封装, 从淘宝上买的源码里应该都有的.

    # first make
    #./make_project.sh R22231 mr706 MR7063H1C6W1 new
    
    # clean and remake
    ./make_project.sh R22231 mr706 MR7063H1C6W1 remake
    
    # remake little kernel
    ./make_project.sh R22231 mr706 MR7063H1C6W1 remake lk
    
    # remake kernel
    ./make_project.sh R22231 mr706 MR7063H1C6W1 remake k bootimage
    
    # remake a module
    ./make_project.sh R22231 mr706 MR7063H1C6W1 mm mediatek/platform/mt6582/hardware/audio

刷机可以用fastboot单刷lk, kernel, system等, 很快, 完整刷一遍则需要5分钟以上, 如果不慎变成砖的话可以用windows上的MTK smart phone flash tool工具来强刷

    # flash little kernel
    adb reboot bootloader; fastboot flash uboot out/target/product/mr706/lk.bin; fastboot reboot
    
    # flash kernel
    adb reboot bootloader; fastboot flash boot out/target/product/mr706/boot.img; fastboot reboot
    
    # flash startup logo
    adb reboot bootloader; fastboot flash logo out/target/product/mr706/logo.bin; fastboot reboot
    
    # flash android system
    adb reboot bootloader; fastboot flash system out/target/product/mr706/system.img; fastboot reboot


### android内核日志查看
    adb logcat
    adb shell cat /proc/sys/kernel/printk
