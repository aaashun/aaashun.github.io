---
layout: post
title: how to recover the removed collection in mongodb
tags: mongodb
---

今天犯二将线上存在mongo里的数据给误删了, 误删的的命令是:

    use foo_db
    db.foo_collection.remove()

查了官方的文档说这种情况不可恢复, stackoverflow上也有不少人说不可恢复, 但根据我对mongodb的了解, mongodb删除记录时并不会立即将数据从磁盘上清理, 而是设一个删除标志位, 运行时被删除的文档是由一个freelist维护, 下次有新数据插入时会先查询freelist以复用这部分空间, 这种思想在不少数据库的实现中都存在的.

由于我及时停掉了mongodb, 保证没有新数据插入, 并将数据文件备份到了安全的位置, 从原理上可断定这部分数据是可找回的。

我写了一个mongorecover小工具成功找回了数据, 具体思路及实现见代码里注释.

https://github.com/ashun/mongorecover
