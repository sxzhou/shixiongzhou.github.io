---
layout: post
title:  "mybatis使用OGNL判断0和空字符串的一点分析"
date:   2018-01-03 23:13:34
categories: mybatis
tags: mybatis
author: "sxzhou"
---  

mapper文件配置如下：  
```java
<if test="status != null and status != ''">status = #{status}</if>
```  

这个字段的判断肯定是copy过来的，`status`本来应该`int`类型的，但被当做字符串判断了，如果`staus>=1`，没有问题，被成功更新了，如果`staus=0`，就出问题了，这个判断是false，也就是说，`OGNL`认为空字符串等于0.  
解决方案：  
```java
<if test="status != null">status = #{status}</if>
```  
或  
```java
<if test="status != null and status != '' or status == 0">status = #{status}</if>
```  
以上是兼容方案，实际上，我们确定了`status`是整数类型，那么就应该判断非`null`且大于0，不应该按照字符串的方式判断。  

底层原因：  
网上有关于这个问题的讨论，但都没有深入探究底层原理，这篇blog有深入到OGNL代码看，但不详细。  
[http://blog.51cto.com/wangguangshuo/1944531](http://blog.51cto.com/wangguangshuo/1944531)  
看了一下OGNL的官网，关于类型转换：  
> Numerical operators try to treat their arguments as numbers. The basic primitive-type wrapper classes (Integer, Double, and so on, including Character and Boolean, which are treated as integers), and the "big" numeric classes from the java.math package (BigInteger and BigDecimal), are recognized as special numeric types. Given an object of some other class, OGNL tries to parse the object's string value as a number.

也就是说，OGNL会把字符串类型转换成数字做比较，转换的规则就是空字符串转换成double值0,因此空字符串就等于0了。  

具体需要深入代码看看什么逻辑，to update......


