---
layout: post
title:  "java集合框架与guava的扩展增强"
date:   2017-12-31 14:55:13
categories: java
tags: guava
author: "sxzhou"
---

`Java Collections Framework`是每个javaer必知必会的，只有深入了解集合框架，才能写出健壮且优雅的代码。当然java集合框架也不是三两句话能讲清楚的，不论是从数据结构，算法还是设计模式，都有许多值得深入研究的地方，除了jdk原生的实现，许多第三方库也对集合框架做了很多扩充，比如，现在公司广泛使用的`guava`。  

## 原生集合框架
google了两张张关于java集合框架的概览图，宏观了解，慢慢消化。  
![](http://7xnwyt.com1.z0.glb.clouddn.com/Java%20Collections%20Framework2.png)  
![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-06-22/Java-collection-cheat-sheet.png)     


对于一些工作中经常使用的集合，收集了一些比较好的blog(有图有真相，品质不错，适合收藏)  

**List**  
[java中的ArrayList,List,LinkedList,Collection关系详解](https://www.cnblogs.com/liqiu/p/3302607.html)  
[极客学院-List总结](http://wiki.jikexueyuan.com/project/java-enhancement/java-thirtytwo.html)  
[Java集合深入理解(4):List<E>接口](http://blog.csdn.net/u011240877/article/details/52802849#)  

**Map**  
[Oracle Java Map集合类简介](http://www.oracle.com/technetwork/cn/articles/maps1-100947-zhs.html)  
[JDK7与JDK8中HashMap的实现](http://www.importnew.com/23164.html)  
[图解HashMap和HashSet的内部工作机制](http://www.importnew.com/21841.html)  
[Java8系列之重新认识HashMap](http://www.importnew.com/20386.html)  
[谈谈HashMap线程不安全的体现](http://www.importnew.com/22011.html)  

**Set**  
[HashSet vs. TreeSet vs. LinkedHashSet](http://www.importnew.com/8773.html)  
[Java HashSet工作原理及实现](http://www.importnew.com/19208.html)  
[如何用Map对象创建Set对象](http://www.importnew.com/9639.html)

**并发集合**  
[java.util.concurrent - Java Concurrency Utilities
](http://tutorials.jenkov.com/java-util-concurrent/index.htm)（很全面但不深入，可以作为目录）  
[ConcurrentHashMap总结](http://www.importnew.com/22007.html)   
[Java并发工具包java.util.concurrent用户指南](http://blog.csdn.net/defonds/article/details/44021605/)  [ConcurrentHashMap原理分析](https://my.oschina.net/hosee/blog/639352)  
[并发队列-无界非阻塞队列 ConcurrentLinkedQueue 原理探究](http://www.importnew.com/25668.html)  
[并发队列 – 无界阻塞队列 LinkedBlockingQueue 原理探究](http://www.importnew.com/22007.html)  
  
## Guava Collections  
Guava对原生集合框架的扩展在四个方面  
#### 1. Immutable collections  
不可变集合从线程安全性，空间利用率，编码安全等方面来说，都有很大的优势，因此，guava给jdk原生集合和guava新增的集合都提供了`Immutable`的实现。如果你没有修改某个集合的需求，或者希望某个集合保持不变时，把它防御性地拷贝到不可变集合是个很好的实践。  
[https://github.com/google/guava/wiki/ImmutableCollectionsExplained](https://github.com/google/guava/wiki/ImmutableCollectionsExplained)  
[http://ifeve.com/google-guava-immutablecollections/](http://ifeve.com/google-guava-immutablecollections/)  
#### 2. New collection types  
包括`Multiset`,`Multimap`,`BiMap`,`Table`,`ClassToInstanceMap`,`RangeMap`,`RangeSet`。在之前的工作中，使用过`Multiset`,`Multimap`做过计数和分组，相当好用，代码简洁功能强大，避免了新建辅助的数据结构或者写出类似`Map<K, Set<V>>`这样不优雅的代码，之前做过一个很复杂的价格区间的处理需求，准备写个线段树处理的，最后发现，`RangeMap`几行代码就搞定了。  
[https://github.com/google/guava/wiki/NewCollectionTypesExplained](https://github.com/google/guava/wiki/NewCollectionTypesExplained)  
[http://ifeve.com/google-guava-newcollectiontypes/](http://ifeve.com/google-guava-newcollectiontypes/)  
#### 3. Utilities  
类似于`java.util.Collections`包含的工具方法。guava对原生集合以及扩展的新集合类提供了更多的工具方法，适用于所有集合的静态方法，比如，平常用得很多的静态工厂方法，支持链式调用的`Iterables`，针对每种不同类型的数据结构，对应的工具类提供了可能用到的各种静态方法，如集合的反转，分割，交并补操作等。  
[https://github.com/google/guava/wiki/CollectionUtilitiesExplained](https://github.com/google/guava/wiki/CollectionUtilitiesExplained)  
[http://ifeve.com/google-guava-collectionutilities/](http://ifeve.com/google-guava-collectionutilities/)  
#### 4. Extension Utilities
扩展集合工具提供了对集合的更高级操作，比如使用装饰器模式做一些个性化操作，实现自己的定制的迭代器等。  
[https://github.com/google/guava/wiki/CollectionHelpersExplained](https://github.com/google/guava/wiki/CollectionHelpersExplained)  
[http://ifeve.com/google-guava-collectionhelpersexplained/](http://ifeve.com/google-guava-collectionhelpersexplained/)  
