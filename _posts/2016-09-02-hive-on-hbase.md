---
layout: post
date: 2016-09-02
title: hive on hbase 的一些注意事项
categories: [hive, hbase]
---
## Hive on Hbase
这段时间一直在搞数据仓库，hive的partition增量，hbase 允许update，允许多job插入同一行数据，都是不可或缺的特点。这就有了hive on hbase的需求。

然而，这个里面有很多坑。本篇blog就列举了一些我遇到的坑。

> 一定要有一行是**:key**

建表时，一定要有一行是 **:key**，且只能出现一次，否则会出现hive表列和hbase表列不一致的错误。

> 除了**:key**的那一列，第一列一定要是所有行都有的qualifier。

建表时，列举的列顺序很重要，除了**:key**之外的第一列一定要是所有行都要有的qualifier。否则，在做query的时候，会过滤掉第一列不存在的那些行。

> hbase中没有得qualifier在hive表中默认值为NULL。

目前还不知道如何解决这个问题。
