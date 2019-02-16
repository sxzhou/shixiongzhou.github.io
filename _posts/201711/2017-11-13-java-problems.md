---
layout: post
title:  "Java常见问题排查(整理自毕玄ppt)"
date:   2017-11-13 09:23:56
categories: article
tags: java
author: "sxzhou"
---

整理自毕玄大神的ppt，结合工作中遇到的相关问题，扩充了部分细节，有一些问题还没有遇到过，整理下来，以备不时之需。   

## NoSuchMethodException  
同类型问题有`ClassNotFoundException`,`NoClassDefFoundError`
#### 原因  
java ClassLoader机制  
jar版本冲突  
#### 排查方法
1. 查看加载的类信息  
    * `-XX:+TraceClassLoading`  
    * `-verbose:class`
    * `jar -tvf *.java`  
2. 检查依赖冲突  
`mvn -U clean package -Dmaven.test.skip=true enforcer:enforce -DcheckDeployRelease_skip=true -Denforcer.skip=false`  
3. 使用maven管理依赖，注意父子POM，正确使用`<dependencies>`与`dependencyManagement`  
4. 使用`<scope>provided</scope>`去除不需要的版本  
[Maven最佳实践](http://juvenshun.iteye.com/blog/337405)    

## ClassNotFoundException   
显示加载一个类通常有如下方式：  
    * 通过`Class`的`forName()`方法；  
    * 通过`ClassLoader`中的`loadClass()`方法；  
    * 通过`ClassLoader`中的`findSystemClass()`方法；  
    
出现这个错误就是当JVM要加载指定文件的字节码到内存时，并没有在指定路径找到这个文件，需要检查当前`classpath`下有没有这个文件，查看当前`classpath`可以使用：  
```java
this.getClass().getClassLoader().getResource("").toString();
```    

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

****
*unable to create new native thread*   
 
#### 原因  
* 线程数超过了`ulimit`,会导致`ps`等出现`resource temporary unavailable`，可以临时调整`ulimit`。  
* 线程数超过了`kernel.pid_max`,只能重启
* 机器完全没有可用内存了  


#### 排查方法  
1. `ps -eLf | grep java -c`  
2. `cat /proc/[pid]/limits`,如果太小，可以调大`max open processes`的值  
3. `sysctl -a | grep kernel.pid_max` 
4. 如果真是线程创建太多，如果有线程名比较好排查，推荐使用线程池，并限制大小，对使用的api要非常清楚，比如很容易误用的netty client    

****

*PermGen Space*  
#### 原因 
PermGen被占满  
#### 排查方法  
1. btrace ClassLoader.defineClass  
2. 确认是否需要装载这么多，如果是，则需要调大`PermGen`，如果不是，需要控制`ClassLoader`，常用于Groovy的误用   

****

*direct buffer memory*  
[堆外内存之 DirectByteBuffer 详解](http://www.importnew.com/26334.html)  
[java之HeapByteBuffer&DirectByteBuffer以及回收DirectByteBuffer](http://blog.csdn.net/xieyuooo/article/details/7547435)
#### 原因  
direct byteBuffer使用超出了限制
#### 解决方法  
1. 调整参数`-XX:MaxDirectMemorySize`  
2. 通常由于网络通信未做限流，需要在业务层做限制    

****
*Map filed*  
#### 原因  
FileChannel mapped文件超出了限制  
#### 排查方法  
1. btrace FileChannel.map  
2. 看看是不是加了`-XX:+DisableExplicitGC`  
#### 解决方法  
1. 如有必要调大`vm.max_map_cout`  
2. 如map file存活个数其实不多，可以去掉`-XX:+DisableExplicitGC`,在CMS GC情况下，增加`-XX:+ExplicitGCInvokesConcurrent`  

****
*request {} bytes for {}. out fo swap space*  
#### 原因   
* 地址空间不够用  
* 物理内存耗光

#### 排查方法  
1. btrace，Deflater.init|Deflater.end|Inflater.init|Inflater.end 
2. 强行执行full gc   

#### 解决方法  
1. 地址空间不够，升级到64位  
2. 物理内存耗光，Inflater/Deflater问题的话显示调用end  
3. DirectByteBuffer问题可以调小-XX:MaxDirectMemorySize
4. 其他case根据google perftools显示的来跟进  

## CPU us高  
#### 原因  
* CMS gc/full gc频繁
* 代码中出现非常耗CPU操作

#### 排查方法  
1. jstack,top待补充  
2. gc log  

## CPU sy高  
#### 原因  
* 锁竞争激烈  
* 线程主动切换频繁
* linux 2.6.32 后的高精度问题   

#### 排查方法   
1. `jstack`看锁状态
2. 是不是有主动线程切换  
3. btrace，AbstractQueuedSynchronizer.ConditionObject.awaitNanos

#### 解决方法
1. 锁竞争激烈，根据业务实现要求合理做锁力度控制，或引入无锁数据结构
2. 线程主动切换，改为通知机制
3. 高精度问题，至少调大到1ms+的await

## CPU iowait高
#### 原因
io读写操作频繁  
#### 排查方法  
1. 确认硬件状况，例如RAID的cache策略
2. 借助系统工具，blktrace+debugfs,iotop,btrace  

#### 解决方法   
1. 提升dirty page cache
2. cache 
3. 同步写转异步写
4. 随机写转顺序写
5. 升级硬件

## java进程退出
#### 原因
原因很多  
#### 排查方法
1. 查看生成的`hs_err_pid[pid].log`
2. 确保`core dump`已打开，`cat /proc/[pid]/limits`
3. `dmesg | grep -i kill` 
4. 根据`core dump`文件分析，`gdb [java路径] core文件`  

## 死锁  
`jstack -l`  
分析线程堆栈信息   

## 线程池耗光  
增加线程数，减少超时时间   

## 调用另一服务超时  
可能原因：  
    * 服务端响应慢  
    * 服务端或客户端GC频繁
    * 服务端或客户端CPU消耗严重
    * 反序列化失败
    * 网络问题   


## 总结
* 知其因
* 唯手熟尔




