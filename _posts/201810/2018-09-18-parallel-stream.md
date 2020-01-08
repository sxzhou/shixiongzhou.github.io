---
layout: post
title:  "When to use parallel streams -- Doug Lea"
date:   2018-10-18 11:00:51
categories: translation
tags: java
author: "sxzhou"
---

原文：
[http://gee.cs.oswego.edu/dl/html/StreamParallelGuidance.html](http://gee.cs.oswego.edu/dl/html/StreamParallelGuidance.html)  

When to use parallel streams
[Draft, 1 September 2014. For now, minimally formatted pending placement decision.]
The java.util.streams framework supports data-driven operations on collections and other sources. Most stream methods apply the same operation to each data element. When multiple cores are available, "data-driven" can become "data-parallel", by using the parallelStream() method of a collection. But when should you do this?

Consider using S.parallelStream().operation(F) instead of S.stream().operation(F) when operations are independent, and either computationally expensive or applied to many elements of efficiently splittable data structures, or both. In more detail:

* F, the per-element function (usually a lambda) is independent: the computation for each element does not rely on or impact that of any other element. (See the stream package summary for further guidance about using stateless non-interfering functions.)
* S, the source collection is efficiently splittable. There are a few other readily parallelizable stream sources besides Collections, for example, java.util.SplittableRandom (for which you can use the stream.parallel() method to parallelize). But most sources based on IO are designed primarily for sequential use.
* The total time to execute the sequential version exceeds a minimum threshold. These days, the threshold is roughly (within a factor of ten of) 100 microseconds across most platforms. You don't need to measure this precisely though. You can estimate this well enough in practice by multiplying N (the number of elements) by Q (cost per element of F), in turn guestimating Q as the number of operations or lines of code, and then checking that N * Q is at least 10000. (If you are feeling cowardly, add another zero or two.) So when F is a tiny function like x -> x + 1, then it would require N >= 10000 elements for parallel execution to be worthwhile. And conversely, when F is a massive computation like finding the best next move in a chess game, the Q factor is so high that N doesn't matter so long as the collection is completely splittable.
The streams framework does not (and cannot) enforce any of these. If the computation is not independent, then running it in parallel will not make any sense and might even be harmfully wrong. The other criteria stem from three engineering issues and tradeoffs:

**Start-up**  
As processors have added cores over the years, most have also added power control mechanisms that can make these cores slow to start up, sometimes with additional overhead imposed by JVMs, OSes, and hypervisors. The threshold roughly approximates the time it might take for enough cores to start processing parallel subtasks to be worthwhile. Once they get started, parallel computations can be more energy efficient than sequential (depending on various processor and system details; see for example this article by Federova et al).  
**Granularity**  
Subdividing already small computations is rarely worthwhile. The framework normally splits up problems so that parts may be processed by all available cores on a system. If there is practically nothing for each core to do after starting, then the (mostly sequential) effort setting up the parallel computation is wasted. Considering that the practical range of cores these days is from 2 to 256, the threshold also stays away from over-partitioning effects.  
**Splittability**  
The most efficiently splittable collections include ArrayLists and {Concurrent}HashMaps, as well as plain arrays (i.e., those of form T[], split using static java.util.Arrays methods). The least efficient are LinkedLists, BlockingQueues, and most IO-based sources. Others are somewhere in the middle. (Data structures tend to be efficiently splittable if they internally support random access, efficient search, or both.) If it takes longer to partition data than to process it, the effort is wasted. So, if the Q factor of computations is high enough, you may get a parallel speedup even for a LinkedList, but this is not very common. Additionally, some sources cannot be split completely down to single elements, so there may be limits in how finely tasks are partitioned.  

Gathering detailed measurements of these effects can be difficult (although possible with some care using tools such as JMH). But overall effects are easy to see. You might experiment yourself to get a feel for them. For example, on a 32-core test machine running tiny functions like max() or sum() on an ArrayList, the break-even is very close to size 10K. Larger sizes see up to a factor of 20 speedup. Run times for sizes less than 10K are not much less than those for 10K, so are often slower than sequential. The worst slowdowns occur when there are fewer than about 100 elements -- this activates a bunch of threads that end up having nothing to do because the computation is finished before they even start. On the other hand, when the per-element computations are time-consuming, benefits are immediate when using an efficiently and completely splittable collection such as an ArrayList.

Another way of saying all this is that using parallel() when there is not enough computation to justify it might cost you around 100 microseconds, and using it when justified should save you at least this much time (possibly hours for very large problems). The exact costs and benefits vary over time and platforms, and also vary across contexts. For example, running a tiny computation in parallel inside a sequential loop accentuates ramp-up and ramp-down effects. (Microbenchmarks that do this might not be predictive about actual usages.)

**Some Questions and Answers**  
1. *Why can't the JVM figure out by itself whether to use parallel mode?*    
It could try, but it would give bad answers far too often. The quest for fully unguided automated multicore parallelism has not been uniformly successful over the past three decades, so the stream framework uses the safer approach of just requiring yes/no decisions by users. These decisions rely on engineering tradeoffs that are unlikely to ever disappear completely, and are similar to those made all the time in sequential programming. For example, you may encounter a factor of a hundred overhead (slowdown) when finding the maximum element in a collection holding only a single element versus just using that element directly (not inside a collection). Sometimes JVMs can optimize away such overhead for you. But this applies infrequently in sequential cases, and never in parallel cases. On the other hand, we do expect that tools will evolve to help users make better decisions.

2. *What if I have too little knowledge about the parameters (F, N, Q, S) to make a good decision?*  
This is also similar to common sequential programming issues. For example calling Collection method S.contains(x) is normally fast if S is a HashSet, slow if a LinkedList, else in-between. Usually, the best way to deal with this is for the author of a component that uses a collection to not export it directly, but instead export operations based on it. Users are then insulated from these decisions. The same applies with parallel operations. For example, a component with an internal Collection "prices" might define a method using a size threshold unless the per-element computations are expensive. For example:
```java
   public long getMaxPrice() { 
       return priceStream().max(); 
    }

   private Stream priceStream() {
     return (prices.size() < MIN_PAR) ? 
        prices.stream() : prices.parallelStream();
    }
```
You can extend this idea in all sorts of ways to handle all sorts of considerations about when and how to use parallelism.

3. *What if my function might do IO or synchronization?*  
At one extreme are functions that don't pass the independence criterion, including intrinsically sequential IO, accesses to locked synchronized resources, and cases in which a failure in one parallel subtask performing IO has side effects on others. Parallelizing these would not make much sense. At the other extreme are computations performing occasional transient IO or synchronization that rarely blocks (for example most forms of logging and most uses of concurrent collections such as ConcurrentHashMap). These are harmless. The in-between cases require the most judgement. If every subtask may be blocked for a significant time waiting for IO or access, CPU resources may sit idle without any way for a program or JVM to use them. Everyone is unhappy. In these cases, parallel streams are not usually a good choice, but good alternatives are available, for example async-IO and CompletableFuture designs.

4. *What if my source is based on IO?*  
Currently, JDK IO-based Stream sources (for example BufferedReader.lines()) are mainly geared for sequential use, processing elements one-by-one as they arrive. Opportunities exist for supporting highly efficient bulk processing of buffered IO, but these currently require custom development of Stream sources, Spliterators, and/or Collectors. Some common forms may be supported in future JDK releases.

5. *What if my program is running on a busy computer and all the cores are being used?*  
Machines generally have only a fixed set of cores, and cannot magically create more when you perform a parallel operation. However, as long as the criteria for choosing parallel execution are clearly met, there is not usually any reason for concern. Your parallel tasks will compete for CPU time with others, so you will see less speed-up. In most cases, this is still more efficient than alternatives. The underlying mechanics are designed such that if no other cores are available, you will see only a small slowdown compared to sequential performance unless the system is already so overloaded that it spends all its time context-switching rather than doing any real work, or is tuned assuming that all processing is sequential. If you are on such a system, the administrator might already have disabled use of multiple threads/cores as part of JVM configuration. And if you are the administrator of such a system, you might consider doing this.

6. *Are all operations parallelized in parallel mode?*  
Yes, at least to some degree, although the Stream framework obeys constraints of the sources and methods in choosing how to do so. In general, fewer constraints enable more potential parallelism. On the other hand, there is no guarantee that the framework will extract and apply all possible opportunities for parallelism. In some cases, if you have the time and expertise, you may be able to hand-craft a significantly better parallel solution.

7. *How much parallel speed-up will I get?*  
If you follow these guidelines, usually enough to be worthwhile. Predictability is not a strong point of modern hardware and systems, so general answers are impossible. Cache locality, garbage-collection rates, JIT compilation, memory contention, data layout, OS scheduling policies, and the presence of hypervisors are among factors that can have significant impact. These also play roles in sequential performance, but are often magnified in parallel settings: An issue causing a ten percent difference in sequential execution might cause a factor of ten difference in parallel.  
The stream framework includes some facilities that can help you improve chances for speedups. For example, using specializations for primitives like IntStream often has a larger effect in parallel than sequential because it not only reduces overhead (and footprint) but also enhances cache locality. And using a ConcurrentHashMap rather than a HashMap as the target of a parallel "collect" operation reduces internal overhead. Further tips and guidance will emerge as people gain experience with the framework.

8. *This is all too scary! Shouldn't we just make a policy of using JVM properties to disable parallelism?*  
We don't want to tell you what to do. Introducing new ways for programmers to get things wrong can be scary. Mistakes in coding, design, and judgement will surely happen. But some people have been predicting for decades that enabling application-level parallelism would lead to major disasters that haven't materialized.  
***
Written by Doug Lea, with the help of Brian Goetz, Paul Sandoz, Aleksey Shipilev, Heinz Kabutz, Joe Bowbeer