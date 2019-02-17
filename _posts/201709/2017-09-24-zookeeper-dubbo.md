---
layout: post
title:  "zookeeper在dubbo中的作用"
date:   2017-09-24 23:11:11
categories: article
tags: zookeeper dubbo java
author: "sxzhou"
---  

公司使用的dubbo强依赖zookeeper，经常在测试环境下，zk挂了，服务起不来，听闻阿里内部使用dubbo是不依赖zk的(待考证)。对于zk在dubbo的作用，第一反应就是服务注册与发现，具体来看。  
![](http://dubbo.io/books/dubbo-user-book/sources/images/zookeeper.jpg)  

工作流程是：  
* 服务提供者启动时: 向 /dubbo/com.foo.BarService/providers 目录下写入自己的 URL 地址  
* 服务消费者启动时: 订阅 /dubbo/com.foo.BarService/providers 目录下的提供者 URL 地址。并向 /dubbo/com.foo.BarService/consumers 目录下写入自己的 URL 地址  
* 监控中心启动时: 订阅 /dubbo/com.foo.BarService 目录下的所有提供者和消费者 URL 地址  

zk的作用：  
* 当提供者出现断电等异常停机时，注册中心能自动删除提供者信息
* 当注册中心重启时，能自动恢复注册数据，以及订阅请求  
* 当会话过期时，能自动恢复注册数据，以及订阅请求  
* 当设置 `<dubbo:registry check="false" />` 时，记录失败注册和订阅请求，后台定时重试
* 可通过 `<dubbo:registry username="admin" password="1234" />` 设置 zookeeper 登录信息
* 可通过 `<dubbo:registry group="dubbo" />` 设置 zookeeper 的根节点，不设置将使用无根树
* 支持 `*` 号通配符 `<dubbo:reference group="*" version="*" />`，可订阅服务的所有分组和所有版本的提供者  

其实，除了zk可以作为dubbo的注册中心，当然也是官方推荐的，dubbo还支持用`redis`和广播来做服务注册。