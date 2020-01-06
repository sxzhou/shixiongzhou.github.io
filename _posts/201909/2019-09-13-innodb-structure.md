---
layout: post
title:  "InnoDB逻辑存储结构"
date:   2019-09-13 19:02:00
categories: reading
tags: java
author: "sxzhou"
---    


## 表空间结构
*《MySQL技术内幕-InnoDB存储引擎(第二版)》*   

![https://s2.ax1x.com/2020/01/06/lyDMHs.jpg](https://s2.ax1x.com/2020/01/06/lyDMHs.jpg)   


从逻辑上看，所有数据都存储在表空间(`Tablespace`)，物理上就是idb文件。   
表空间又可以逐级细分为：段(`Segment`)，区(`Extent`)，页(`Page`)，行(`Row`)。   

1. 表空间(`Tablespace`)   
所有数据存放的逻辑最高层次。
2. 段(`Segment`)
表空间由各个段组成，常见的有数据段、索引段、回滚段。InnoDB是索引组织表，数据段就是聚集索引B+树的叶子节点，索引段就是非叶子节点。
3. 区(`Extent`)
每个区大小是1MB，默认情况下页的大小是`16KB`，那么一个区中就包含64个连续的页。为保证页的连续性，一般一次从磁盘申请4到5个区。
4. 页(`Page`)   
页是InnoDB管理的`最小单位`，默认16KB，包括数据页、undo页、系统页等。

5. 行(`Row`)
InnoDB是以行为基础管理数据的。也就是数据按行存放的。每一页存储的行记录是有限制的，最多`16K/2-200`行记录，也就是`7992`行。

## 索引结构  
InnoDB是索引组织表，主键索引B+树的叶子节点存储数据，非主键索引B+树叶子节点存储主键，因此通过非主键索引查询，需要先找到主键，然后回表通过主键索引查到数据。   
宏观来看主键索引与非主键索引：
![https://s2.ax1x.com/2020/01/06/ly2azq.png](https://s2.ax1x.com/2020/01/06/ly2azq.png)   

微观来看主键索引树的page： 
![https://s2.ax1x.com/2020/01/06/ly24OK.png](https://s2.ax1x.com/2020/01/06/ly24OK.png)
