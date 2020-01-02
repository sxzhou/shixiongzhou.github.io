---
layout: post
title:  "同步IO/异步IO/阻塞IO/非阻塞IO"
date:   2019-04-20 19:02:00
categories: article
tags: java
author: "sxzhou"
---

对于后端开发来说，这几个名词经常能听到，但是具体怎么解释，还真不是那么好说。比如，异步IO就是非阻塞IO吗？  
***   
## **1. 一些观点**  
来自知乎[相关问题](https://www.zhihu.com/question/19732473/answer/20851256)高票回答。   
1. **同步和异步**关注的是消息通信机制 ，所谓同步，就是在发出一个调用时，在没有得到结果之前，该调用就不返回。但是一旦调用返回，就得到返回值了。**阻塞和非阻塞**关注的是程序在等待调用结果（消息，返回值）时的状态.
阻塞调用是指调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会返回。非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程。   
2. 来自C++大神陈硕的解释：在处理 IO 的时候，阻塞和非阻塞都是同步 IO。只有使用了特殊的 API 才是异步 IO。
![https://s2.ax1x.com/2020/01/02/lNJsdP.jpg](https://s2.ax1x.com/2020/01/02/lNJsdP.jpg)   

来自[StackOverflow](https://stackoverflow.com/questions/2625493/asynchronous-vs-non-blocking)：  
>Blocking and synchronous mean the same thing: you call the API, it hangs up the thread until it has some kind of answer and returns it to you.   
Non-blocking means that if an answer can't be returned rapidly, the API returns immediately with an error and does nothing else. So there must be some related way to query whether the API is ready to be called (that is, to simulate a wait in an efficient way, to avoid manual polling in a tight loop).   
Asynchronous means that the API always returns immediately, having started a "background" effort to fulfil your request, so there must be some related way to obtain the result.   
***
## **2. Linux IO模型**  
具体到Linux环境下，说到IO模型，就不得不提Richard Stevens的*UNIX网络编程*，大致分为以下几种：  
### **2.1 Blocking IO**  
![https://s2.ax1x.com/2020/01/02/lNtLr9.jpg](https://s2.ax1x.com/2020/01/02/lNtLr9.jpg)   
当用户进程调用了recvfrom这个系统调用，kernel就开始了IO的第一个阶段：准备数据。对于network io来说，很多时候数据在一开始还没有到达（比如，还没有收到一个完整的UDP包），这个时候kernel就要等待足够的数据到来。而在用户进程这边，整个进程会被阻塞。当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态，重新运行起来。   

### **2.2 Non-blocking IO**   
![https://s2.ax1x.com/2020/01/02/lNUNTI.jpg](https://s2.ax1x.com/2020/01/02/lNUNTI.jpg)  
当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。
所以，用户进程其实是需要不断的主动询问kernel数据好了没有。   

### **2.3 IO multiplexing**   
![https://s2.ax1x.com/2020/01/02/lNU6mj.jpg](https://s2.ax1x.com/2020/01/02/lNU6mj.jpg)  
当用户进程调用了select，那么整个进程会被block，而同时，kernel会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read操作，将数据从kernel拷贝到用户进程。
这个图和blocking IO的图其实并没有太大的不同，事实上，还更差一些。因为这里需要使用两个system call (select 和 recvfrom)，而blocking IO只调用了一个system call (recvfrom)。但是，用select的优势在于它可以同时处理多个connection。  

### **2.4 Asynchronous I/O**  
![https://s2.ax1x.com/2020/01/02/lNdkqJ.jpg](https://s2.ax1x.com/2020/01/02/lNdkqJ.jpg)  
用户进程发起read操作之后，立刻就可以开始去做其它的事。而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了。  


通过上面的图示，可以对比下非阻塞IO和异步IO，还是区别很大的，在non-blocking IO中，虽然进程大部分时间都不会被block，但是它仍然要求进程去主动的check，并且当数据准备完成以后，也需要进程主动的再次调用recvfrom来将数据拷贝到用户内存。而asynchronous IO则完全不同。它就像是用户进程将整个IO操作交给了他人（kernel）完成，然后他人做完后发信号通知。在此期间，用户进程不需要去检查IO操作的状态，也不需要主动的去拷贝数据。    
