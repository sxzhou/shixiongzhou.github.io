---
layout: post
title:  "使用xxd分析文本中的不可见字符"
date:   2017-08-23 23:00:00
categories: linux
tags: tools
author: "sxzhou"
---

今天执行一个简单的sql，总是提示有语法错误，可以确定sql的语法是没有问题的，只可能是sql文本字符串有问题，使用`xxd`这个二进制文件查看工具，它可以将文件以十六进制对应输出。　　
简化复现一下，原始文件invisible.txt:  
```sql
select 
orderId, price
from orders
where 
id = 12345
```
执行命令
`xxd invisible.txt`  
输出:  
```
00000000: 7365 6c65 6374 200a 6f72 6465 7249 642c  select .orderId,
00000010: 2070 7269 6365 0a66 726f 6d20 6f72 6465   price.from orde
00000020: 7273 0a77 6865 7265 c2a0 0a69 6420 3d20  rs.where...id = 
00000030: 3132 3334 350a                           12345. 
```
可以看到`where`后面除了换行符`0a`前面还有不可见字符，查询utf-8字符编码表:  
[http://www.utf8-chartable.de/unicode-utf8-table.pl?utf8=0x](http://www.utf8-chartable.de/unicode-utf8-table.pl?utf8=0x)  
发现`0xc2 0xa0`这个code point对应的字符是`NO-BREAK SPACE`，[不换行空格](https://zh.wikipedia.org/wiki/%E4%B8%8D%E6%8D%A2%E8%A1%8C%E7%A9%BA%E6%A0%BC)是一个空格字符，用途是禁止自动换行。HTML页面显示时会自动合并多个连续的空白字符（whitespace character），但该字符是禁止合并的，因此该字符也称作“硬空格”（hard space、fixed space）。  
这个字符导致sql异常，可以将它删除:  
`tr -d "\302\240" < invisible.txt > visible.txt`  

#### 参考　　
[http://www.utf8-chartable.de/unicode-utf8-table.pl?utf8=0x](http://www.utf8-chartable.de/unicode-utf8-table.pl?utf8=0x)  
[https://zh.wikipedia.org/wiki/UTF-8](https://zh.wikipedia.org/wiki/UTF-8)  
[http://www.voidcn.com/article/p-fthfycuu-mk.html](http://www.voidcn.com/article/p-fthfycuu-mk.html)  
[http://xiangwangfeng.com/2011/05/02/UTF-8%E8%BD%ACGBK%E7%9A%84%E6%82%B2%E5%89%A7-%E7%89%B9%E6%AE%8A%E5%AD%97%E7%AC%A6C2A0/](http://xiangwangfeng.com/2011/05/02/UTF-8%E8%BD%ACGBK%E7%9A%84%E6%82%B2%E5%89%A7-%E7%89%B9%E6%AE%8A%E5%AD%97%E7%AC%A6C2A0/)


