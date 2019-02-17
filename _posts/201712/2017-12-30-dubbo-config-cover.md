---
layout: post
title:  "dubbo配置覆盖策略"
date:   2017-12-30 12:19:52
categories: article
tags: java dubbo
author: "sxzhou"
---

最近发现有个服务超时，超过了默认配置的`timeout`，调整这个参数，顺便整理下dubbo的配置覆盖策略。  
简而言之，根据dubbo官方文档，读取配置的查找顺序可以概括为:  
* 方法级优先，接口级次之，全局配置最次
* 级别一样，则消费方优先，提供方次之

如何理解，以`timeout`这个参数的配置为例，如果以xml方式配置:  
![](http://dubbo.io/books/dubbo-user-book/sources/images/dubbo-config-override.jpg)  

上图中，对于`timeout`的取值，优先级依次递减，如果都没有配置，取默认配置，`timeout`的默认配置是1s。  

对于很多公共的配置，dubbo还提供了配置文件和JVM启动参数的方式，这三种配置方式的优先级是：  
* JVM启动参数最优  
* xml配置次之  
* properties配置文件最次

以dubbo服务端口号为例：  
![](http://dubbo.io/books/dubbo-user-book/sources/images/dubbo-properties-override.jpg)  

[dubbo官方文档](http://dubbo.io/books/dubbo-user-book/)