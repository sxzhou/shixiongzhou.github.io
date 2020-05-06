---
layout: post
title:  "Java的线程与协程"
date:   2020-04-30 19:02:00
categories: reading
tags: java
author: "sxzhou"
---    
## 1. 线程实现的几种方式
线程是比进程更轻量级的调度执行单元，线程的引入可以把一个进程的资源分配和执行调度分开，线程可以共享进程资源，如内存、文件IO等，又可以独立调度使用CPU资源。  
### 1.1 使用内核线程   
**内核线程**：Kernel Level Thread（KLT），是由操作系统内核直接支持的线程，内核通过调度器讲线程任务映射到处理器上。  
**轻量级进程**： Light Weight Process（LWP），内核线程的高级接口，用户进程通常不会直接使用内核线程，而是使用LWP提供的接口实现线程功能，每一个轻量级进程都对应一个内核线程，这是一种`1:1`的线程模型。  
![thread1](https://s1.ax1x.com/2020/05/05/YF6Ftf.png)  

在现在常用的Oracle JDK和Linux操作系统环境下，使用的是这种模型。  

### 1.2 使用用户线程实现  
**用户线程**：User Thread（UT），通常来看，用户线程就是完全建立在用户空间的线程库上，对操作系统内核无感知，用户线程的建立、同步、销毁和调度完全在用户态中完成。线程切换不需要到内核态操作，因此操作可以是非常快速且低消耗的，也可以支持规模更大的线程数量。这种线程模型是`1:N`的。  
![thread2](https://s1.ax1x.com/2020/05/05/YF6C7t.png)  

由于用户线程的调度完全由用户程序控制，因此实现比较复杂，Java历史版本曾经使用过这种模型但最终放弃。  

### 1.3 使用用户线程加轻量级进程混合实现  
这种模式下，既存在用户线程，也存在轻量级进程。用户线程还是完全建立在用户空间中，因此用户线程的创建、切换、析构等操作依然廉价，并且可以支持大规模的用户线程并发。而操作系统提供支持的轻量级进程则作为用户线程和内核线程之间的桥梁，这样可以使用内核提供的线程调度功能及处理器映射，并且用户线程的系统调用要通过轻量级线程来完成。  
这种模式下，用户线程和轻量级进程的比例是不固定的，称为`M:N`模型。
![thread3](https://s1.ax1x.com/2020/05/05/YF6iAP.png)

## 2. Java多线程编程  
这是个很宽泛的概念，包含了线程、线程池、线程安全等很多相关课题，相关资料和书籍很多，作为后端开发人员应该有比较深入的理解。放一篇美团技术团队关于线程池的文章，顺便说下，美团技术团队的文章干货很多，值得借鉴。  
[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)  

## 3. Java线程模型存在的问题  
如1.1提到的，每个轻量级进程都对应一个内核线程，首先，由于是基于内核线程实现的，所以各种线程操作，如创建、析构及同步，都需要进行系统调用。而系统调用的代价相对较高，需要在用户态和内核态中来回切换。其次，每个轻量级进程都需要有一个内核线程的支持，因此轻量级进程要消耗一定的内核资源（如内核线程的栈空间），因此一个系统支持轻量级进程的数量是有限的。在64位的操作系统环境中，JVM 会为每个线程分配1M的栈空间，这在高并发环境下内存空间占用是很恐怖的，同时存在上下文切换，当内核从一个线程切换至另一个线程时，有很多的工作要做。新运行的线程和进程必须要将其他线程也在同一个 CPU 上运行的事实抽象出去，通常情况下上下文切换的消耗可以忽略不计，但是在高并发场景下，会造成CPU资源被大量浪费在上下文切换。  
更多讨论可以参考：  

[为什么能有上百万个 Goroutines，却只能有上千个 Java 线程？](https://www.infoq.cn/article/a-million-go-routines-but-only-1000-java-threads)  

[为什么 Java 坚持多线程不选择协程？](https://www.zhihu.com/question/332042250)   

## 4. Java的协程  
Java目前没有协程的语言层面的实现，但是有相关的开源框架实现类似的目的，包括Quasar、Project Loom、阿里巴巴开源的AJDK，基本原理都是都是通过字节码操作将同步调用改为异步调用。  
[https://github.com/puniverse/quasar](https://github.com/puniverse/quasar)  
[https://openjdk.java.net/projects/loom/](https://openjdk.java.net/projects/loom/)  
[Java 异步编程：从 Future 到 Project Loom](https://www.jianshu.com/p/5db701a764cb)（这篇文章总结不错，从Future模式到反应式编程再到Project Loom各种方案的优缺点。）  

但是这些项目并没有广泛推广使用和验证，然而兄弟语言Kotlin从语言层面支持了Coroutine，值得了解下。  
[KotlinConf 2017 - Introduction to Coroutines by Roman Elizarov](https://www.youtube.com/watch?v=_hfBv0a09Jc)  
[了解Kotlin协程实现原理这篇就够了](https://ethanhua.github.io/2018/12/24/kotlin_coroutines/)
