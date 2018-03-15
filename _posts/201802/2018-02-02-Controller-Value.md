---
layout: post
title:  "springMVC在Controller中使用@Value注入属性失败原因分析"
date:   2018-02-02 23:00:34
categories: spring
tags: spring
author: "sxzhou"　　
---　　

这个问题的关键在于：  
* ApplicationContext和WebApplicationContext是什么关系？
* property文件放在哪里？是谁加载的？

**ApplicationContext**  
官方解释：  
> The interface org.springframework.context.ApplicationContext represents the Spring IoC container and is responsible for instantiating, configuring, and assembling the aforementioned beans. The container gets its instructions on what objects to instantiate, configure, and assemble by reading configuration metadata. The configuration metadata is represented in XML, Java annotations, or Java code.  

ApplicationContext代表spring的IoC容器，加载方式是ContextLoaderListener和ContextLoaderServlet，通过web.xml的如下配置：  
```java
<listener>
     <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<context-param>
     <param-name>contextConfigLocation</param-name>
     <param-value>classpath:*-context.xml</param-value>
</context-param>
```  
**WebApplicationContext**  
官方解释：  
> In the Web MVC framework, each DispatcherServlet has its own WebApplicationContext, which inherits all the beans already defined in the root WebApplicationContext. These inherited beans can be overridden in the servlet-specific scope, and you can define new scope-specific beans local to a given Servlet instance.  

每一个DispatcherServlet都会有一个WebApplicationContext，通常我们的web应用只配置一个WebApplicationContext。通过web.xml配置：  
```java
<servlet>
      <servlet-name>platform-services</servlet-name>
      <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
      <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:platform-services-servlet.xml</param-value>
      </init-param>
      <load-on-startup>1</load-on-startup>
</servlet>
```  
WebApplicationContext可以引用ApplicationContext加载的bean，反之不行。  

**@Value如何加载**  
通过BeanPostProcessor加载，BeanPostProcessor的说明：  
> BeanPostProcessor interfaces are scoped per-container. This is only relevant if you are using container hierarchies. If you define a BeanPostProcessor in one container, it will only do its work on the beans in that container. Beans that are defined in one container are not post-processed by a BeanPostProcessor in another container, even if both containers are part of the same hierarchy.  

可以看到，BeanPostProcessor是每个context一个，不能共享，因此，如果配置文件加载是在ApplicationContext，那么在WebApplicationContext中是无法获取的。  

Stackoverflow上有几个涉及到多个容器的问题：  
[https://stackoverflow.com/questions/11890544/spring-value-annotation-in-controller-class-not-evaluating-to-value-inside-pro?rq=1](https://stackoverflow.com/questions/11890544/spring-value-annotation-in-controller-class-not-evaluating-to-value-inside-pro?rq=1)  
[https://stackoverflow.com/questions/3652090/difference-between-applicationcontext-xml-and-spring-servlet-xml-in-spring-frame](https://stackoverflow.com/questions/3652090/difference-between-applicationcontext-xml-and-spring-servlet-xml-in-spring-frame)  
[https://stackoverflow.com/questions/11890544/spring-value-annotation-in-controller-class-not-evaluating-to-value-inside-pro?rq=1](https://stackoverflow.com/questions/11890544/spring-value-annotation-in-controller-class-not-evaluating-to-value-inside-pro?rq=1)  

他们都引用了这篇博客：  
[http://www.codesenior.com/en/tutorial/Spring-ContextLoaderListener-And-DispatcherServlet-Concepts](http://www.codesenior.com/en/tutorial/Spring-ContextLoaderListener-And-DispatcherServlet-Concepts)  
逻辑很清晰，找时间翻译一下。

