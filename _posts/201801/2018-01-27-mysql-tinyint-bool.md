---
layout: post
title:  "Mysql中TINYINT(1)自动映射成Boolean解决方法"
date:   2018-01-28 00:10:11
categories: article
tags: mysql
author: "sxzhou"
---  

之前碰到过两次这个问题，总结一下：  
mysql中没有bool类型，如果把一个字段设置为bool类型，mysql自动把他转换为tinyint(1)存储，反之，tinyint(1)类型的字段值如果是1，那么就会映射成true，如果是0则映射成false。  
解决方案：  
1. 数据库连接串配置加上`tinyInt1isBit=false`
2. 既然mysql对tinyint(1)特殊对待，那么不要使用tinyint(1)，位宽设置为其他值  
3. `select vaule*1`，对返回的字段做一次强制的计算转换成整型

