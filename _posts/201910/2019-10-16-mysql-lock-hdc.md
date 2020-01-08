---
layout: post
title:  "MySQL加锁处理分析(何登成的博客)"
date:   2019-10-16 19:02:00
categories: reading
tags: mysql
author: "sxzhou"
---   

记得刚毕业参加应届生培训时，老师给我们讲MySQL基础，对于一些很深入的地方，老师让我们好好看看何登成的博客，并且在工作中再慢慢琢磨理解。现在有幸和何登成在同一家公司，在内网也看了他的文章，回看他的分享依然很多收获，但发现他的[个人博客](http://hedengcheng.com/?p=771)很久没更新了，图也挂了，在网上找了些残存记录收藏起来。

---   

    1. 背景

     1.1    MVCC：Snapshot Read vs Current Read

     1.2    Cluster Index：聚簇索引

     1.3    2PL：Two-Phase Locking    

     1.4    Isolation Level    

    2.    一条简单SQL的加锁实现分析    

     2.1    组合一：id主键+RC    

     2.2    组合二：id唯一索引+RC    

     2.3    组合三：id非唯一索引+RC    

     2.4    组合四：id无索引+RC    

     2.5    组合五：id主键+RR    

     2.6    组合六：id唯一索引+RR    

     2.7    组合七：id非唯一索引+RR    

     2.8    组合八：id无索引+RR    

     2.9    组合九：Serializable   

    3. 一条复杂的SQL    

    4. 死锁原理与分析    

    5. 总结    

---   

### **1. 背景**   
MySQL/InnoDB的加锁分析，一直是一个比较困难的话题。我在工作过程中，经常会有同事咨询这方面的问题。同时，微博上也经常会收到MySQL锁相关的私信，让我帮助解决一些死锁的问题。本文，准备就MySQL/InnoDB的加锁问题，展开较为深入的分析与讨论，主要是介绍一种思路，运用此思路，拿到任何一条SQL语句，都能完整的分析出这条语句会加什么锁？会有什么样的使用风险？甚至是分析线上的一个死锁场景，了解死锁产生的原因。

 

注：MySQL是一个支持插件式存储引擎的数据库系统。本文下面的所有介绍，都是基于InnoDB存储引擎，其他引擎的表现，会有较大的区别。   

#### 1.1 MVCC：Snapshot Read vs Current Read   

MySQL InnoDB存储引擎，实现的是基于多版本的并发控制协议——MVCC (Multi-Version Concurrency Control) (注：与MVCC相对的，是基于锁的并发控制，Lock-Based Concurrency Control)。MVCC最大的好处，相信也是耳熟能详：读不加锁，读写不冲突。在读多写少的OLTP应用中，读写不冲突是非常重要的，极大的增加了系统的并发性能，这也是为什么现阶段，几乎所有的RDBMS，都支持了MVCC。

 

在MVCC并发控制中，读操作可以分成两类：快照读 (snapshot read)与当前读 (current read)。快照读，读取的是记录的可见版本 (有可能是历史版本)，不用加锁。当前读，读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。

 

在一个支持MVCC并发控制的系统中，哪些读操作是快照读？哪些操作又是当前读呢？以MySQL InnoDB为例：   

* 快照读：简单的select操作，属于快照读，不加锁。(当然，也有例外，下面会分析)   

```sql
    select * from table where ?;
```
* 当前读：特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。所有以下的语句，都属于当前读，读取记录的最新版本。并且，读取之后，还需要保证其他并发事务不能修改当前记录，对读取记录加锁。其中，除了第一条语句，对读取记录加S锁 (共享锁)外，其他的操作，都加的是X锁 (排它锁)。

```sql
    select * from table where ? lock in share mode;
    select * from table where ? for update;
    insert into table values (…);
    update table set ? where ?;
    delete from table where ?;
```
   
              
为什么将 插入/更新/删除 操作，都归为当前读？可以看看下面这个 更新 操作，在数据库中的执行流程：    
![1](https://upload-images.jianshu.io/upload_images/3596546-08017b39ae25ac41.png?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)   

从图中，可以看到，一个Update操作的具体流程。当Update SQL被发给MySQL后，MySQL Server会根据where条件，读取第一条满足条件的记录，然后InnoDB引擎会将第一条记录返回，并加锁 (current read)。待MySQL Server收到这条加锁的记录之后，会再发起一个Update请求，更新这条记录。一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。因此，Update操作内部，就包含了一个当前读。同理，Delete操作也一样。Insert操作会稍微有些不同，简单来说，就是Insert操作可能会触发Unique Key的冲突检查，也会进行一个当前读。

注：根据上图的交互，针对一条当前读的SQL语句，InnoDB与MySQL Server的交互，是一条一条进行的，因此，加锁也是一条一条进行的。先对一条满足条件的记录加锁，返回给MySQL Server，做一些DML操作；然后在读取下一条加锁，直至读取完毕。

#### 1.2 Cluster Index：聚簇索引       
InnoDB存储引擎的数据组织方式，是聚簇索引表：完整的记录，存储在主键索引中，通过主键索引，就可以获取记录所有的列。关于聚簇索引表的组织方式，可以参考MySQL的官方文档：Clustered and Secondary Indexes 。本文假设读者对这个，已经有了一定的认识，就不再做具体的介绍。接下来的部分，主键索引/聚簇索引 两个名称，会有一些混用，望读者知晓。   

#### 1.3 2PL：Two-Phase Locking   

传统RDBMS加锁的一个原则，就是2PL (二阶段锁)：Two-Phase Locking。相对而言，2PL比较容易理解，说的是锁操作分为两个阶段：加锁阶段与解锁阶段，并且保证加锁阶段与解锁阶段不相交。下面，仍旧以MySQL为例，来简单看看2PL在MySQL中的实现。   
![2](https://upload-images.jianshu.io/upload_images/3596546-51a1b79d25e499b7.png?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)   

从上图可以看出，2PL就是将加锁/解锁分为两个完全不相交的阶段。加锁阶段：只加锁，不放锁。解锁阶段：只放锁，不加锁。   


#### 1.4 Isolation Level    
隔离级别：Isolation Level，也是RDBMS的一个关键特性。相信对数据库有所了解的朋友，对于4种隔离级别：Read Uncommited，Read Committed，Repeatable Read，Serializable，都有了深入的认识。本文不打算讨论数据库理论中，是如何定义这4种隔离级别的含义的，而是跟大家介绍一下MySQL/InnoDB是如何定义这4种隔离级别的。

 

MySQL/InnoDB定义的4种隔离级别：       
* Read Uncommited    
可以读取未提交记录。此隔离级别，不会使用，忽略。

* Read Committed (RC)    
快照读忽略，本文不考虑。针对当前读，RC隔离级别保证对读取到的记录加锁 (记录锁)，存在幻读现象。

* Repeatable Read (RR)   
快照读忽略，本文不考虑。针对当前读，RR隔离级别保证对读取到的记录加锁 (记录锁)，同时保证对读取的范围加锁，新的满足查询条件的记录不能够插入 (间隙锁)，不存在幻读现象。

* Serializable   
从MVCC并发控制退化为基于锁的并发控制。不区别快照读与当前读，所有的读操作均为当前读，读加读锁 (S锁)，写加写锁 (X锁).Serializable隔离级别下，读写冲突，因此并发度急剧下降，在MySQL/InnoDB下不建议使用。    

### 2.    一条简单SQL的加锁实现分析    

在介绍完一些背景知识之后，本文接下来将选择几个有代表性的例子，来详细分析MySQL的加锁处理。当然，还是从最简单的例子说起。经常有朋友发给我一个SQL，然后问我，这个SQL加什么锁？就如同下面两条简单的SQL，他们加什么锁？

 

SQL1：
```sql
select * from t1 where id = 10;
```    

SQL2：
```sql
delete from t1 where id = 10;
```

针对这个问题，该怎么回答？我能想象到的一个答案是：

SQL1：不加锁。因为MySQL是使用多版本并发控制的，读不加锁。
SQL2：对id = 10的记录加写锁 (走主键索引)。
 

这个答案对吗？说不上来。即可能是正确的，也有可能是错误的，已知条件不足，这个问题没有答案。如果让我来回答这个问题，我必须还要知道以下的一些前提，前提不同，我能给出的答案也就不同。要回答这个问题，还缺少哪些前提条件？

 

前提一：id列是不是主键？
前提二：当前系统的隔离级别是什么？
前提三：id列如果不是主键，那么id列上有索引吗？
前提四：id列上如果有二级索引，那么这个索引是唯一索引吗？
前提五：两个SQL的执行计划是什么？索引扫描？全表扫描？
 

没有这些前提，直接就给定一条SQL，然后问这个SQL会加什么锁，都是很业余的表现。而当这些问题有了明确的答案之后，给定的SQL会加什么锁，也就一目了然。下面，我将这些问题的答案进行组合，然后按照从易到难的顺序，逐个分析每种组合下，对应的SQL会加哪些锁？

 

注：下面的这些组合，我做了一个前提假设，也就是有索引时，执行计划一定会选择使用索引进行过滤 (索引扫描)。但实际情况会复杂很多，真正的执行计划，还是需要根据MySQL输出的为准。

 

* 组合一：id列是主键，RC隔离级别
* 组合二：id列是二级唯一索引，RC隔离级别
* 组合三：id列是二级非唯一索引，RC隔离级别
* 组合四：id列上没有索引，RC隔离级别
* 组合五：id列是主键，RR隔离级别
* 组合六：id列是二级唯一索引，RR隔离级别
* 组合七：id列是二级非唯一索引，RR隔离级别
* 组合八：id列上没有索引，RR隔离级别
* 组合九：Serializable隔离级别
 

排列组合还没有列举完全，但是看起来，已经很多了。真的有必要这么复杂吗？事实上，要分析加锁，就是需要这么复杂。但是从另一个角度来说，只要你选定了一种组合，SQL需要加哪些锁，其实也就确定了。接下来，就让我们来逐个分析这9种组合下的SQL加锁策略。

 

注：在前面八种组合下，也就是RC，RR隔离级别下，SQL1：select操作均不加锁，采用的是快照读，因此在下面的讨论中就忽略了，主要讨论SQL2：delete操作的加锁。   

#### 2.1    组合一：id主键+RC  

这个组合，是最简单，最容易分析的组合。id是主键，Read Committed隔离级别，给定SQL：delete from t1 where id = 10; 只需要将主键上，id = 10的记录加上X锁即可。如下图所示：
#### 2.2    组合二：id唯一索引+RC    
#### 2.3    组合三：id非唯一索引+RC    
#### 2.4    组合四：id无索引+RC    
#### 2.5    组合五：id主键+RR    
#### 2.6    组合六：id唯一索引+RR    
#### 2.7    组合七：id非唯一索引+RR    
#### 2.8    组合八：id无索引+RR    
#### 2.9    组合九：Serializable   

 
