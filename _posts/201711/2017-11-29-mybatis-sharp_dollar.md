---
layout: post
title:  "Mybatis中#和$的区别"
date:   2017-11-29 22:34:33
categories: article
tags: mybatis
author: "sxzhou"
---  

在mybatis中的$与#都是在sql中动态的传入参数。区别在于：  
* \#{}: 解析为一个 JDBC 预编译语句（prepared statement）的参数标记符，一个 #{ } 被解析为一个参数占位符 。  
* ${}: 仅仅为一个纯碎的 string 替换，在动态 SQL 解析阶段将会进行变量替换。  

[官方文档解释](http://www.mybatis.org/mybatis-3/sqlmap-xml.html#Parameters)：  
> By default, using the #{} syntax will cause MyBatis to generate PreparedStatement properties and set the values safely against the PreparedStatement parameters (e.g. ?). While this is safer, faster and almost always preferred, sometimes you just want to directly inject an unmodified string into the SQL Statement. For example, for ORDER BY, you might use something like this:  <br>
ORDER BY ${columnName} <br>
Here MyBatis won't modify or escape the string. <br>
NOTE <br>
It's not safe to accept input from a user and supply it to a statement unmodified in this way. This leads to potential SQL Injection attacks and therefore you should either disallow user input in these fields, or always perform your own escapes and checks.

所以，使用$是单纯的字符串替换，有sql注入风险，大部分情况，我们应该使用#传入变量，如果是order by这种字符串替换，可以使用$，效率更高。  
