---
layout: post
title:  "Java常见问题排查(整理自毕玄ppt)"
date:   2017-11-13 09:23:56
categories: java
tags: java
author: "sxzhou"
---

整理自毕玄大神的ppt，结合工作中遇到的相关问题，扩充了部分细节，有一些问题还没有遇到过，整理下来，以备不时之需。
***
## NoSuchMethodException  
同类型问题有ClassNotFoundException,NoClassDefFoundError
#### 原因  
java ClassLoader机制  
jar版本冲突  
#### 排查方法
1. 查看加载的类信息  
    * -XX:+TraceClassLoading  
    * -verbose:class
    * jar -tvf *.java  
2. 检查依赖冲突  
`mvn -U clean package -Dmaven.test.skip=true enforcer:enforce -DcheckDeployRelease_skip=true -Denforcer.skip=false`  
3. 使用maven管理依赖，注意父子POM，正确使用`<dependencies>`与`dependencyManagement`  
4. 使用`<scope>provided</scope>`去除不需要的版本  
[Maven最佳实践](http://juvenshun.iteye.com/blog/337405)  
***
## ClassNotFoundException  
显示加载一个类通常有如下方式：  
    * 通过Class的forName()方法；  
    * 通过ClassLoader中的loadClass()方法；  
    * 通过ClassLoader中的findSystemClass()方法；  
出现这个错误就是当JVM要加载指定文件的字节码到内存时，并没有在指定路径找到这个文件，需要检查当前classpath下有没有这个文件，查看当前classpath可以使用：  
```java
this.getClass().getClassLoader().getResource("").toString();
```  
***
## 死锁  
`jstack -l`  
分析线程堆栈信息  
***
## 线程池耗光  
增加线程数，减少超时时间  
***
## 调用另一服务超时  
可能原因：  
    * 服务端响应慢  
    * 服务端或客户端GC频繁
    * 服务端或客户端CPU消耗严重
    * 反序列化失败
    * 网络问题
***
## OOM   

*GC overhead limit exceed/java heap space*  
#### 原因  
java heap分配不出需要的内存了  
TODO tu  
#### 排查方法  
  1. 确定系统内存占用  
  `free -m`
  2. 拿到heap dump文件  
  `-XX:+HeapDumpOnOutOfMemoryError`  
  `jmap -dump:file=<file name>,format=b [pid]`  
  `gcore [pid]`    
  3. 分析dump文件(MAT)  
  [Java程序内存分析：使用mat工具分析内存占用](http://developer.51cto.com/art/201407/444487.htm)  
  [MAT - Memory Analyzer Tool 使用进阶](http://ju.outofmemory.cn/entry/204399)  
  4. 根据MAT分析结果定位代码(btrace)  
  [BTrace工具简介](http://mgoann.iteye.com/blog/1409667)
  
  [一次线上OOM过程的排查](http://blog.csdn.net/qq_16681169/article/details/53296137)  
  类似的问题还有full gc频繁。  
***  
*unable to create new native thread*  
#### 原因  
* 线程数超过了ulimit,会导致ps等出现resource temporary unavailable，可以临时调整ulimit。  
* 线程数超过了kernel.pid_max,只能重启
* 机器完全没有可用内存了  
#### 排查方法  
1. ps -eLf | grep java -c  
2. cat /proc/[pid]/limits,如果太小，可以调大max open processes的值  
3. sysctl -a | grep kernel.pid_max  
4. 如果真是线程创建太多，如果有线程名比较好排查，推荐使用线程池，并限制大小，对使用的api要非常清楚，比如很容易误用的netty client  
***  
*PermGen Space*  
#### 原因 
PermGen被占满  
#### 排查方法  
1. btrace ClassLoader.defineClass  
2. 确认是否需要装载这么多，如果是，则需要调大PermGen，如果不是，需要控制ClassLoader，常用于Groovy的误用  
***  
*direct buffer memory*  
[堆外内存之 DirectByteBuffer 详解](http://www.importnew.com/26334.html)  
[java之HeapByteBuffer&DirectByteBuffer以及回收DirectByteBuffer](http://blog.csdn.net/xieyuooo/article/details/7547435)
#### 原因  
direct byteBuffer使用超出了限制
#### 解决方法  
1. 调整参数`-XX:MaxDirectMemorySize`  
2. 通常由于网络通信未做限流，需要在业务层做限制  
***  
*Map filed*  
#### 原因  
FileChannel mapped文件超出了限制  
#### 排查方法  
1. btrace FileChannel.map  
2. 看看是不是加了`-XX:+DisableExplicitGC`  
#### 解决方法  
1. 如有必要调大`vm.max_map_cout`  
2. 如map file存活个数其实不多，可以去掉`-XX:+DisableExplicitGC`,在CMS GC情况下，增加`-XX:+ExplicitGCInvokesConcurrent`  
***  



