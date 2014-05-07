---
layout: post
title: 简单的nosqlite实现
comments: true
tags: nosql
---

这段时间在优化内核字典, 字典存储的内容是25万个左右单词的发音字典, 之前是用sqlite存储的, 占空间在80MB左右, 传输过程中使用zip -9压缩后也还有16MB. 这对一些特殊设备来说占的存储空间太大, 且下载过程需要更多流量. 另外在一些嵌入式微操作系统上没法直接使用sqlite.

其实我需要的只是一个简单的文件型kv存储, 使用时独占的, 只有查询, 能达到1000qps, 内存占用要小就行了. 估摸着自已写一个, 代码量应该在500行以内.

存储格式:

    header
    klen1 key1 vlen1 value1
    klen2 key2 vlen2 value2

header占12字节, 标明版本号. klen占一个字节, klen的第一个bit是删除标记位, 后7个bit表示实际大小, 这样如果klen大于127说明这条记录被删除了, 紧接着是key, key后面存储vlen和value, vlen占两个字节, 所以记录大小最大为64KB

打开数据库的过程是将key做一遍索引保存在hash表里, hash node结构如下:

    struct node {
        unsigned int hash2;
        unsigned int pos;
        struct node *next;
    };

hash2是采用另一种hash算法计算出的hash值, 这样是为了节约内存, 避免在内存里保存key内容, 这可能会产生冲突, 没关系, 二次hash的情况下冲突的机率是2^32 * capacity, 如果发现冲突调大capacity就行了. pos是klen在数据文件中的位置, 这样可精确知道某条记录在文件中的位置, 两次fseek和fread就可读出文件内容.

支持的操作:

* set - 插入的过程就是按上边的格式append文件就行了
* get - 查找的过程是先从hash表中取得记录pos, 再从数据文件中读取记录内容
* remove - 删除的过程是从hash表中删除hash node并取得记录pos, 从pos位置读出klen, 置删除标记即可(klen+128再写回)

一些测试数据:

机器配置: E5200 2.50GHz, SATA 7200

* 测试25万个单词的词表的实际加载时间: 130ms
* 测试25万个单词的词表在sqlite上的随机qps: 24798
* 测试25万个单词的词表在simpledb上的随机qps: 411149
* 内存测试: total heap usage: 41 allocs, 41 frees, 4,026,576 bytes allocated

目前适用场景:

* 百万级记录(应该更高, 我没做额外的测试)
* 每个记录在64K以内
* 数据文件在2G以内
* 键大小在127个字节以内
* 一个数据文件同时被多进程打开时只能读(后续版本可支持多进程读写)

我把它起名叫nosqlite, 已经放到<a href="https://github.com/ashun/nosqlite">github</a>上了
