---
layout: post
title: 让adb识别android设备
comments: true
tags: android
---

今天解决一个问题需要用到小米2手机, 折腾了好一会, 才在'adb devices'里看到设备.
这前遇到过一次这类问题, 其实解决开发环境无法识别android设备的方法都差不多, 这里记录一下完整过程.

<h2>0. 首先确认你的usb线是支持数据传输的</h2><br/>
曾经一次折腾了半天发现是只是根充电线有问题, 插上能正常充电, 但是lsusb看不到

<h2>1. 获取USB Vendor ID</h2><br/>
断开设备执行lsusb, 连接设备再执行lsusb, 根据结果可判断设备的idVender, 如下输出中2717即是小米2的vendor
<pre>
    Bus 005 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
    Bus 004 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
    Bus 003 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
    Bus 002 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
    Bus 001 Device 003: ID 2717:9039
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
</pre><br/>


<h2>2. 修改udev rules</h2><br/>
<pre>
    echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="2717", MODE="0666", GROUP="plugdev"' >> /etc/udev/rules.d/51-android.rules
    chmod a+r /etc/udev/rules.d/51-android.rules
</pre><br/>


<h2>3. 修改.android/adb_usb.ini</h2><br/>
如果是三星,htc等大厂的设备不需要此步, 建议在这个列表里找不到的都设一下吧, http://developer.android.com/tools/device.html
<pre>
    echo '0x2717' >> ~/.android/adb_usb.ini
</pre><br/>


<h2>4. 重启 udev</h2><br/>
<pre>
    #service udev restart # ubuntu
    /sbin/start_udev # centos
</pre></br>


<h2>5. 重启 adb server</h2></br>
<pre>
    adb kill-server
    adb start-server
    adb devices
</pre></br>


<h2>6. 如果adb devices还不显示则可能是adb的版本号太低</h2></br>
执行adb version查看版本号, 目前最新版本号是1.0.31
<pre>
    ${ANDROID_SDK}/tools/android update adb
</pre>

重复步骤4即可


参考:
  http://developer.android.com/tools/device.html
