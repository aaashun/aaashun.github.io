---
layout: post
title: using aimake to build libiconv for android
tags: aimake
---
今日一同事在android项目中用到iconv, 需要编译一个android版的静态库, 折腾了好久也没编译成功. 直接使用ndk编译libiconv关键是配置好交叉编译文件(Android.mk), 这里给出一份更简单的方法.

将如下内容保存在aimakefile
<pre>
mk := 'LOCAL_MODULE := iconv\n\n'
mk += 'LOCAL_CFLAGS := -DHAVE_CONFIG_H -DBUILDING_LIBICONV -DBUILDING_DLL -DENABLE_RELOCATABLE=1 -DIN_LIBRARY -DNO_XMALLOC -DLIBDIR=\"\" -Dset_relocation_prefix=libiconv_set_relocation_prefix -Drelocate=libiconv_relocate -I .  -I include -I libcharset/include -I lib -I libcharset/lib\n\n'
mk += 'LOCAL_SRC_FILES := lib/iconv.c libcharset/lib/localcharset.c lib/relocatable.c\n\n'
mk += 'include $$(BUILD_STATIC_LIBRARY)\n\n'
mk += 'install:\n'
mk += '\tcp include/iconv.h $$(PLATFORM)/usr/include\n'
mk += '\tcp libiconv.a $$(PLATFORM)/usr/lib'
all:
&#9;@rm -rf libiconv*
&#9;@echo "downloading libiconv ..."
&#9;@wget -q -O /dev/stdout http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.14.tar.gz | tar -xzf /dev/stdin
&#9;@mv libiconv-* libiconv
&#9;@echo -e $(mk) > libiconv/aimakefile
&#9;@sed -i -e 's/-e //' -e 's/^ //' libiconv/aimakefile
&#9;@cd libiconv; ./configure;
&#9;@sed -i '/HAVE_LANGINFO_CODESET/s/1/0/' libiconv/config.h
&#9;@sed -i '/HAVE_LANGINFO_CODESET/s/1/0/' libiconv/lib/config.h
&#9;@sed -i '/HAVE_LANGINFO_CODESET/s/1/0/' libiconv/libcharset/config.h
&#9;@cd libiconv; aimake -t android clean all install;
</pre><br/>

编译android平台的libiconv, 执行完如下命令会在libiconv目录下生成android版本的libiconv.a, 并且自动将iconv.h和libiconv.a安装到android-ndk/platforms/android-3/arch-arm目录下
<pre>
aimake
</pre>


aimake是基于gnu make的一个简单编译工具, 支持ios, android, linux, darwin, mingw32, 等平台的编译和交叉编译. https://github.com/ashun/aimake

----
参考:

* http://blog.sina.com.cn/s/blog_82f2fc28010133k1.html
