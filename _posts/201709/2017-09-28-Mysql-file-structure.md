---
layout: post
title:  "Mysql(InnoDB)表空间存储结构"
date:   2017-09-28 23:14:45
categories: article
tags: mysql InnoDB
author: "sxzhou"
---  

![](https://images.cnblogs.com/cnblogs_com/arlen/innodb_tablespace.jpg)  

![](https://images.cnblogs.com/cnblogs_com/arlen/tablespace.jpg)  

数据段即B+树的叶子节点，索引段即为B+树的非叶子节点
InnoDB存储引擎的管理是由引擎本身完成的，表空间（Tablespace）是由分散的段(Segment)组成。一个段(Segment)包含多个区（Extent）。
区（Extent）由64个连续的页（Page）组成，每个页大小为16K，即每个区大小为1MB，创建新表时，先使用32页大小的碎片页存放数据，使用完后才是区的申请（InnoDB最多每次申请4个区，保证数据的顺序性能）

[https://dev.mysql.com/doc/refman/5.5/en/innodb-file-space.html](https://dev.mysql.com/doc/refman/5.5/en/innodb-file-space.html)
