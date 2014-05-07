---
layout: post
title: 获取android设备的唯一ID
comments: true
tags: android
---

为了对使用我的基础库的客户进行授权需要对每个Android设备进行唯一标识, 一路翻阅下来发现可选择的标识真多, 且每个标识都有些小遗憾。

在Android开发者官方blog上已经有一篇文章对此做了总结(参考链接1), 这里结合自已查询的资料再总结一下, 并给出最终符合要求的解决方案。

<h2>1. ANDROID_ID, Secure.ANDROID_ID </h2><br/>
但是据说Motorola的Droid2犯了个低级错误, 所有的Droid2上使用同一个ANDROID_ID, 即9774D56D682E549C, ANDROID_ID的内部实现是存储在系统的一个SQLite数据库里, root过就有权限修改了

<h2>2. TelephonyManager.getDeviceId()</h2><br/>
返回IMEI, MEID 或 ESN, 需要READ_PHONE_STATE权限

在过去, 那时所有的Android设备都是电话, 这个方法足够用了, 但是现在连Android电视也有了

<h2>3. Serial Number, android.os.Build.SERIAL</h2><br/>
在Android 2.3以后的版本, 非手机的设备必须设置此序列号, 有些手机也设置此序列号

<h2>4. Mac Address</h2><br/>
WiFi或Bluetooth设备的网卡物理地址, 但是并不是所有Android设备都有WiFi或Bluetooth, 且在WiFi没打开时也不能正常获取

<h2>5. cpu serial</h2><br/>
CPU序列号, 通过cat /proc/cpuinfo可以查询, 我的Sumsung能正确查询, 但在有的设备上查询出的序列号是0

我的实现是优先使用ANDROID_ID, 再次是imei, 最后是android.os.Build.SERIAL

在java层的实现如下(java代码):

{% highlight java %}
String deviceId;
String androidId = Secure.getString(getBaseContext().getContentResolver(), Secure.ANDROID_ID);
String imei = ((TelephonyManager) getBaseContext().getSystemService(Context.TELEPHONY_SERVICE )).getDeviceId();
String serial = android.os.Build.SERIAL;

deviceId = androidId;
if (deviceId == null || "9774d56d682e549c".equals(deviceId)) {
    deviceId = imei;
    if (deviceId == null || "".equals(deviceId)) {
        deviceId = serial;
        if (deviceId == null || deviceId.equals("") || deviceId.equals("unknown")) {
            deviceId = null;
        }
    }
}

Log.d(TAG, "ANDROID-ID: " + androidId);
Log.d(TAG, "imei: " + imei);
Log.d(TAG, "SERIAL: " + serial);
Log.d(TAG, "final deviceId: " + deviceId);
{% endhighlight %}

在NDK层的实现就比较困难了, 脱离了java层的Context对象, 就无法获取ANDROID_ID和DeviceID了, 这里我增加了一个接口参数, 在调用时传入一个Context对象进来

在NDK层的实现如下(c代码):

{% highlight c %}
int
get_android_device_id(char device_id[64], JNIEnv *jenv, jobject jcontext)
{
    static char _device_id[64];
    const char *android_id, *imei, *serial;
    jstring     jandroid_id, jimei, jserial;

    if (!jenv || !jcontext) {
        goto end;
    }

    /* String androidId = android.provider.Settings.Secure.getString(getApplicationContext().getContentResolver(), android.provider.Settings.Secure.ANDROID_ID); */
    /* String imei = ((android.telephony.TelephonyManager) getApplicationContext().getSystemService(android.content.Context.TELEPHONY_SERVICE)).getDeviceId(); */
    /* String serial = android.os.Build.SERIAL; */

    jandroid_id = (*jenv)->CallStaticObjectMethod(jenv,
            (*jenv)->FindClass(jenv, "android/provider/Settings$Secure"),
            (*jenv)->NewStringUTF(jenv, "android_id"));

    jimei = (*jenv)->CallObjectMethod(jenv,
            (*jenv)->CallObjectMethod(jenv, jcontext, (*jenv)->GetMethodID(jenv, (*jenv)->GetObjectClass(jenv, jcontext), "getSystemService", "(Ljava/lang/String;)Ljava/lang/Object;"), (*jenv)->NewStringUTF(jenv, "phone")),
    jimei = (*jenv)->GetMethodID(jenv, (*jenv)->FindClass(jenv, "android/telephony/TelephonyManager"), "getDeviceId", "()Ljava/lang/String;"));

    jserial = (*jenv)->GetStaticObjectField(jenv,
            (*jenv)->FindClass(jenv, "android/os/Build"),
            (*jenv)->GetStaticFieldID(jenv, (*jenv)->FindClass(jenv, "android/os/Build"), "SERIAL", "Ljava/lang/String;"));

    android_id = jandroid_id ? (*jenv)->GetStringUTFChars(jenv, jandroid_id, NULL) : NULL;
    imei       = jimei ? (*jenv)->GetStringUTFChars(jenv, jimei, NULL) : NULL;
    serial     = jserial ? (*jenv)->GetStringUTFChars(jenv, jserial, NULL) : NULL;

    if (android_id && strcmp(android_id, "") && strcmp(android_id, "9774d56d682e549c")) {
        strcpy(_device_id, android_id);
    } else if (imei && strcmp(imei, "")) {
        strcpy(_device_id, imei);
    } else if  (serial && strcmp(serial, "")) {
        strcpy(_device_id, serial);
    } else {
        strcpy(_device_id, "");
    }

    if (strlen(_device_id) <= 10) {
        strcpy(_device_id, "");
    }

    (*jenv)->ReleaseStringUTFChars(jenv, jandroid_id, android_id);
    (*jenv)->ReleaseStringUTFChars(jenv, jimei, imei);
    (*jenv)->ReleaseStringUTFChars(jenv, jserial, serial);

end:
    if (device_id) {
        strcpy(device_id, _device_id);
    }

    return 0;
}
{% endhighlight %}

JNI代码写起来很啰嗦, 写这段代码时对着手册看了半天

----

参考:

* http://android-developers.blogspot.com/2011/03/identifying-app-installations.html
* http://support.verizonwireless.com/clc/devices/knowledge_base.html?id=26468
* https://groups.google.com/forum/?fromgroups#!topic/android-developers/U4mOUI-rRPY
* https://groups.google.com/forum/?fromgroups#!topic/android-developers/8bIp6VERRQk
* What is exactly an ANDROID_ID, https://groups.google.com/forum/?fromgroups#!topic/android-developers/Rn15F7Ku4GM
* Pseudo-Unique ID, www.pocketmagic.net/2011/02/android-unique-device-id/#.UcMfVreA9_s
* Get ANDROID-ID in NDK, https://groups.google.com/forum/?fromgroups#!topic/android-ndk/0AiE_ohuUjM
* http://stackoverflow.com/questions/12103411/android-ndk-crash-in-callobjectmethod-calling-getsystemservice
