---
layout: post
title:  "使用sql parser处理sql文本"
date:   2019-09-26 19:02:00
categories: article
tags: java
author: "sxzhou"
---    

最近项目中需要一个简单的数据查询平台，数据量比较大存储在在线olap数据库和离线大数据平台(类hive)中，支持可视化配置查询条件，或者直接提交sql文本，流程浓缩起来类似于：   
`(可视化配置条件)` -> `sql` -> `DB引擎`   

sql生成工具比较多，推荐`jOOQ`提供的sql builder，`jOOQ`现在很活跃，官方文档丰富，而且作者及公司员工常年关注`stackoverflow`上的相关问题，项目中碰到问题问过几次，很快就会回复：   

[https://stackoverflow.com/questions/57591155/how-to-generate-sql-from-template-with-order-by-parameter-using-jooq](https://stackoverflow.com/questions/57591155/how-to-generate-sql-from-template-with-order-by-parameter-using-jooq)  
[https://stackoverflow.com/questions/57601416/how-to-modify-plain-sql-text-with-jooq-such-as-appending-order-by-limit-offset-c](https://stackoverflow.com/questions/57601416/how-to-modify-plain-sql-text-with-jooq-such-as-appending-order-by-limit-offset-c)  
[https://stackoverflow.com/questions/57707116/how-to-parser-sql-string-with-self-defined-functions-using-jooq](https://stackoverflow.com/questions/57707116/how-to-parser-sql-string-with-self-defined-functions-using-jooq)   

`jOOQ`提供的sql builder非常好用，但是sql parser还不是很完善，最初以为sql parser的使用应该不会太多，做些语法校验，加limit等，但随着项目深入发现一个好的sql parser是必须的。   


最终选用了`druid sql parser`(内部加强版)，这是经过阿里线上应用检验过得，性能没有问题，其实更重要的是支持公司内部数据库方言，这一点至关重要。   

开源版本文档：   
[https://github.com/alibaba/druid/wiki/SQL-Parser](https://github.com/alibaba/druid/wiki/SQL-Parser)   

sql parser的基本执行流程：   
![https://s2.ax1x.com/2020/01/11/lIc6l8.png](https://s2.ax1x.com/2020/01/11/lIc6l8.png)   

三个核心模块：   
* `Parser`    
`parser`是将输入文本转换为`AST`（抽象语法树），`parser`有包括两个部分，`parser`和`lexer`，其中`lexer`实现词法分析，`parser`实现语法分析。
* `AST`  
`AST`是`Abstract Syntax Tree`的缩写，也就是抽象语法树。`AST`是`parser`输出的结果。  
* `Visitor`   
`Visitor`是遍历AST的手段，是处理`AST`最方便的模式，`Visitor`是一个接口，有缺省什么都没做的实现`VistorAdapter`。   

把sql文本解析成`AST`，同时使用`Visitor`，就能方便的获取sql信息以及改写sql了。   

关于`druid sql parser`的详细分析，除了官方文档，`segmentfault`上的几篇文章很不错。   
[Druid SQL 解析器概览](https://segmentfault.com/a/1190000008120215)   
[Druid SQL 解析器的解析过程](https://segmentfault.com/a/1190000008120254)  
