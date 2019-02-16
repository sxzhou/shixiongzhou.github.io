---
layout: post
title:  "Mysql应用最佳实践总结(updating)"
date:   2017-11-30 09:54:10
categories: article
tags: mysql
author: "sxzhou"
---  

从Mysql应用层面总结的一些最佳实践经验(注意存储引擎是InnoDB),分别从表字段设计，数据库范式，索引，explain，sql禁忌，事务等方面总结。    

#### 1. 表字段设计  
* 表需要一个业务无关的自增ID作为主键，命名为id，数据类型是UNSIGNED，指定AUTO_INCREMENT。  
      [https://ruby-china.org/topics/26352](https://ruby-china.org/topics/26352)  
* 大字段，低频字段分表存放，冷热数据分离  
* 合理使用char，varchar，text，blob，尽量不要使用text，blob。  
  [http://www.cnblogs.com/billyxp/p/3548540.html](http://www.cnblogs.com/billyxp/p/3548540.html)  
  [http://hidba.org/?p=551](http://hidba.org/?p=551)  
* 尽量不要使用ENUM，使用TINYINT更通用   
  [http://www.neatstudio.com/show-1498-1.shtml](http://www.neatstudio.com/show-1498-1.shtml)
* 正确理解INT(4)的含义(位宽与实际长度)  
* 涉及货币,金融等数值，使用DECIMAL代替FLOAT和DOUBLE  


#### 2. 范式  
现在数据库设计最多满足3NF，普遍认为范式过高，虽然具有对数据关系更好的约束性，但也导致数据关系表增加而令数据库IO更易繁忙，原来交由数据库处理的关系约束现更多在数据库使用程序中完成。  
[http://blog.csdn.net/yangbodong22011/article/details/51619590](http://blog.csdn.net/yangbodong22011/article/details/51619590)  
[http://blog.wuxu92.com/hp-mysql-paradigm/](http://blog.wuxu92.com/hp-mysql-paradigm/)  

#### 3. 索引  
* 联合索引最左匹配原则的应用
* 使用覆盖索引优化查询
* join两边的字段需要添加索引
* 索引不在多而在精，避免过度索引
* 尽量避免更新索引列值，设计索引重建
* 在选择性高的字段添加索引
* order by的字段尽量添加索引，避免file sort
* 线上查询不能无索引可走
* 哪些情况无法使用索引
  * where条件没有限制
  * 否定条件：<>,not in,not exist
  * join连接字段类型或字符集不一致
  * 扫描范围超过全表20%
  * where字段使用了函数
  * like '%xxx'
  * 隐式字符类型转换



#### 4. Explain  


#### 5. 一些sql禁忌  
* select/delete/update无where条件或条件范围太大
* 避免使用子查询，子查询产生的临时表无索引可走
* 不要再索引字段上使用函数
* 避免select *，无法使用覆盖索引，浪费io资源  


#### 6. 正确使用事务
* 避免出现太大的事务操作，容易出现死锁，锁超时，主从数据不一致
* 事务中不要出现RPC
* 多事务写表注意顺序，防止死锁
* 事务传播级别为PROPAGATION_REQUIRES_NEW时，注意子事务不要操作主事务锁住的记录