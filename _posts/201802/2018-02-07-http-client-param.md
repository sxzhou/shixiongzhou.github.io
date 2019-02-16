---
layout: post
title:  "HttpClient连接数参数解析"
date:   2018-02-07 13:11:23
categories: article
tags: web HttpClient
author: "sxzhou"
---  
节前高峰，被百度导了一波流量过来，qps突然升高了1000+，调用第三方的HttpClient抛异常，获取不到连接，赶紧调整了限流参数，查看连接池参数配置，这个并发数应该可以支持的啊，为什么会崩呢？查看了参数说明，果然，使用了公司提供的默认配置，最大连接数确实不够。之前对连接数参数的理解也是有误的。  

对于连接数的配置，我们常用以下三个指标：  
* the total number of connections  
* the maximum number of connections per (any) route
* the maximum number of connections per a single, specific route  

根据api说明：  
* setMaxTotal(int max): Set the maximum number of total open connections.
* setDefaultMaxPerRoute(int max): Set the maximum number of concurrent connections per route, which is 2 by default.
* setMaxPerRoute(int max): Set the total number of concurrent connections to a specific route, which is 2 by default.  

如下配置：  
```java
PoolingHttpClientConnectionManager connManager 
  = new PoolingHttpClientConnectionManager();
connManager.setMaxTotal(200);
connManager.setDefaultMaxPerRoute(50);
HttpHost host = new HttpHost("www.baidu.com", 80);
connManager.setMaxPerRoute(new HttpRoute(host), 30);
```  
以上配置表示，整个连接池的最大连接数是200，默认每个路由的最大连接数是50，对于baidu这个路由最大连接数是30。这里需要注意，对一个特定的host，最大连接数限制真正起作用的是DefaultMaxPerRoute和MaxPerRoute，总连接数不超过MaxTotal。  

#### 参考  
[http://www.baeldung.com/httpclient-connection-management](http://www.baeldung.com/httpclient-connection-management)  
[http://jinnianshilongnian.iteye.com/blog/2089792](http://jinnianshilongnian.iteye.com/blog/2089792)