---
layout: post
title:  "一个有意思的bug-正确使用并行流"
date:   2018-10-10 11:00:51
categories: article
tags: java8
author: "sxzhou"
---    

使用Java8的Stream API处理集合元素确实比较方便，最近碰到个问题，由于一个map操作涉及IO操作比较耗时，从以前多线程开发经验看，可以考虑扩展下并行度，但是事与愿违，并未有提高处理效率。   

于是在StackOverflow问了下各路大神，详细问题如下：
[https://stackoverflow.com/questions/52287717/how-to-specify-forkjoinpool-for-java-8-parallel-stream](https://stackoverflow.com/questions/52287717/how-to-specify-forkjoinpool-for-java-8-parallel-stream)  

总结起来实际上这是两个问题：
#### 1. 怎样自定义ForkJoinPool   

[https://stackoverflow.com/questions/21163108/custom-thread-pool-in-java-8-parallel-stream](https://stackoverflow.com/questions/21163108/custom-thread-pool-in-java-8-parallel-stream)  

这个问题下有详细说明：

>For applications that require separate or custom pools, a ForkJoinPool may be constructed with a given target parallelism level; by default, equal to the number of available processors.  

>This also means if you have nested parallel streams or multiple parallel streams started concurrently, they will all share the same pool.    
Advantage:   
>you will never use more than the default (number of available processors).       
Disadvantage:    
>you may not get "all the processors" assigned to each parallel stream you initiate (if you happen to have more than one). (Apparently you can use a ManagedBlocker to circumvent that.)   

>To change the way parallel streams are executed,    
you can either     
>>submit the parallel stream execution to your own ForkJoinPool: yourFJP.submit(() -> stream.parallel().forEach(soSomething)).get(); 

>or  
>>you can change the size of the common pool using system properties: System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "20") for a target parallelism of 20 threads. However, this no longer works after the backported patch https://bugs.openjdk.java.net/browse/JDK-8190974.   

也就是默认情况下，使用的是默认的ForkJoinPool（即ForkJoinPool.commonPool()）,默认并行度是当前CPU核数。  
如果需要改变，有两种选择：1. 自定义ForkJoinPool，2. 修改`java.util.concurrent.ForkJoinPool.common.parallelism`参数，调整默认ForkJoinPool的并行度。为了处理这个特定场景下的问题，选用第一种方式，也就是自定义ForkJoinPool。   

```java
@Test
public void stream() throws Exception {
    //System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "20");
    ForkJoinPool pool = new ForkJoinPool(10);
    List<Integer> testList = Lists.newArrayList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20);
    long start = System.currentTimeMillis();
    List<Integer> result = pool.submit(() -> testList.parallelStream().map(item -> {
        try {
            // read from database
            Thread.sleep(1000);
            System.out.println("task" + item + ":" + Thread.currentThread());
        } catch (Exception e) {
        }
        return item * 10;
    })).get().collect(Collectors.toList());
    System.out.println(result);
    System.out.println(System.currentTimeMillis() - start);
}
```   

然而并没有解决，于是还有第二个问题。   


#### 2. Stream操作的“懒执行”   

>Streams are lazy; all work is done when you commence a terminal operation. In your case, the terminal operation is .collect(Collectors.toList()), which you call in the main thread on the result of get(). Therefore, the actual work will be done the same way as if you’ve constructed the entire stream in the main thread.  

于是按照`Holger`的回答改成如下，并行效果达到了。

```java
ForkJoinPool pool = new ForkJoinPool(10);
List<Integer> testList = Arrays.asList(
    1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20);
long start = System.currentTimeMillis();
List<Integer> result = pool.submit(() -> testList.parallelStream().map(item -> {
    try {
        // read from database
        Thread.sleep(1000);
        System.out.println("task" + item + ":" + Thread.currentThread());
    } catch (InterruptedException e) {}
    return item * 10;
}).collect(Collectors.toList())).join();
System.out.println(result);
System.out.println(System.currentTimeMillis() - start);
```
参考`java8实战`第80页，核心点在于，流包含三个部分：
* 一个数据源（如集合）来执行查询
* 一个中间操作链，形成一条流水线
* 一个终端操作，生成结果   

执行的流程是：
>Intermediate operations return a new stream. They are always lazy; executing an intermediate operation such as filter() does not actually perform any filtering, but instead creates a new stream that, when traversed, contains the elements of the initial stream that match the given predicate. Traversal of the pipeline source does not begin until the terminal operation of the pipeline is executed.   

碰到`terminal operation`才会真正执行，`intermediate operation`只是产生一个新的流，如果`terminal operation`在main线程，那么map操作默认使用的是default ForkJoinPool。  

![https://s2.ax1x.com/2020/01/08/lRuT54.png](https://s2.ax1x.com/2020/01/08/lRuT54.png)

关于“懒执行”解释可以参考：   
[https://www.logicbig.com/tutorials/core-java-tutorial/java-util-stream/lazy-evaluation.html](https://www.logicbig.com/tutorials/core-java-tutorial/java-util-stream/lazy-evaluation.html)

