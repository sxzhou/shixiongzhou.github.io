---
layout: post
title:  "springboot配置多数据源的坑"
date:   2017-08-25 23:00:00
categories: spring
tags: spring
author: "sxzhou"
---

使用springboot配置多数据源时，碰到一个坑，搜了一下网上的解决方案，都是说其中一个数据源要加注解@Primary，才能启动，但是都没有讲清楚原因。  
stackoverflow上看到类似一个说清楚了，记录一下。

springboot文档有：  
> Spring JDBC has a DataSource initializer feature. Spring Boot enables it by default and loads SQL from the standard locations schema.sql and data.sql (in the root of the classpath).  

也就是springboot对数据源有一个初始化的机制，确保可以正常访问配置的数据源，单一数据源没有问题，如果配置了多个数据源，在初始化`org.springframework.boot.autoconfigure.jdbc.DataSourceInitializer`时有以下逻辑：  
```java
@PostConstruct
public void init() {
    if (!this.properties.isInitialize()) {
        logger.debug("Initialization disabled (not running DDL scripts)");
        return;
    }
    if (this.applicationContext.getBeanNamesForType(DataSource.class, false, false).length > 0) {
        ==> this.dataSource = this.applicationContext.getBean(DataSource.class);
    }
    if (this.dataSource == null) {
        logger.debug("No DataSource found so not initializing");
        return;
    }
    runSchemaScripts();
}
```  
问题就出在`applicationContext.getBean(DataSource.class)`这一步，看方法的定义：  
><T> T getBean(Class<T> requiredType) throws BeansException 
Return the bean instance that uniquely matches the given object type, if any. 
This method goes into ListableBeanFactory by-type lookup territory but may also be translated into a conventional by-name lookup based on the name of the given type. For more extensive retrieval operations across sets of beans, use ListableBeanFactory and/or BeanFactoryUtils.   <br> Parameters: 
requiredType - type the bean must match; can be an interface or superclass. null is disallowed.  <br>
Returns:   <br>
an instance of the single bean matching the required type  <br>
Throws: <br>
NoSuchBeanDefinitionException - if no bean of the given type was found 
==> NoUniqueBeanDefinitionException - if more than one bean of the given type was found 
BeansException - if the bean could not be created    

因此，如果有多个bean符合，会抛出异常，导致启动失败。  
解决方法也很简单：  
* 如果你需要初始化数据源这个功能，那么指定其中一个数据源@Primary  
* 如果不需要这个功能，可以配置springboot不作数据源初始化检查，在application.properties配置spring.datasource.initialize=false  

#### 参考  
https://stackoverflow.com/questions/44757388/why-qualifier-not-work/44776477?noredirect=1#comment76540296_44776477
  
