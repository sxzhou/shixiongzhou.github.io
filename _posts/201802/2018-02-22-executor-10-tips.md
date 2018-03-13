---
layout: post
title:  "[译]使用ExecutorService的10个技巧"
date:   2018-02-22 23:09:23
categories: translation
tags: concurrency
author: "sxzhou"
---  

#### 1. 为线程池中的线程命名  
这点再多强调都不为过，不管是从JVM中dump出正在执行的线程，或是调试，可以看到，默认的线程命名模式是`pool-N-thread-M`，其中，`N`表示线程池的序列号(每次新建一个线程池，全局的计数器N就会加1)，`M`表示线程池内部线程的序列号。比如，`pool-2-thread-3`就表示，这是当前JVM生命周期内，第二个线程池中的创建的第三个线程，描述性太差了。如果使用JDK中提供的方法给线程池命名，会比较麻烦，因为命名规则藏在`ThreadFactory`里。幸好，Guava给我们提供了一个帮助类，可以自定义线程命名模式：  
```java
import com.google.common.util.concurrent.ThreadFactoryBuilder;
final ThreadFactory threadFactory = new ThreadFactoryBuilder()
.setNameFormat("Orders-%d")
.setDaemon(true)
.build();
final ExecutorService executorService = Executors.newFixedThreadPool(10, threadFactory);
```  
使用这个帮助类需要注意，默认情况下，创建的线程是守护线程，你需要确定是否符合你的要求。  

#### 2. 根据上下文切换线程名  
这个技巧我是从[ Supercharged jstack: How to Debug Your Servers at 100mph](http://www.takipiblog.com/supercharged-jstack-how-to-debug-your-servers-at-100mph/)这篇文章学到的，一旦我们记录了线程名，那么在运行时我们就可以随意的更改它，这点很重要，因为线程dump出来会展示类和方法的名字，但没有参数和本地变量相关的信息，通过在线程名中记录一些必要的信息，我们可以方便的跟踪这些信息，便于排查定位慢查询或死锁的。  
例如：  
```java
private void process(String messageId) {
executorService.submit(() -> {
final Thread currentThread = Thread.currentThread();
final String oldName = currentThread.getName();
currentThread.setName("Processing-" + messageId);
try {
//real logic here...
} finally {
currentThread.setName(oldName);
}
});
}
```  
在`try-finally`代码块中，当前执行的线程会命名为`Processing-WHATEVER-MESSAGE-ID-IS`，这对于从线程日志信息中定位问题是很方便的。  

#### 3. 显式且安全地关闭线程池  
在线程池中，除了正在执行的线程，还有一个等待的任务队列，当你的应用关闭的时候，你需要考虑两件事情：队列中等待的任务会怎样？正在执行的线程会怎样(这个稍后详述)？令人吃惊的是很多开发者并没有意识到这个问题并合理的关闭线程池。对于这个问题，有两种方式：要么让所有等待的任务执行完(shutdown())或丢弃它们(shutdownNow()),具体要看你的使用场景。例如，我们提交了一堆任务，想要等到它们都执行完毕就关闭线程池，那么就需要用shutdown():  
```java
private void sendAllEmails(List<String> emails) throws InterruptedException {
emails.forEach(email ->
executorService.submit(() ->
sendEmail(email)));
executorService.shutdown();
final boolean done = executorService.awaitTermination(1, TimeUnit.MINUTES);
log.debug("All e-mails were sent so far? {}", done);
}
```  
在这个示例中，我们使用线程池发送一堆邮件，发送每封邮件都是一个单独的任务，等提交了所有的任务后，我们就关闭了线程池，此后线程池不会再接受新的任务。然后我们等待所有的线程执行完毕，最多等待一分钟，如果一分钟后还有任务没有执行完，那么`awaitTermination`会简单的返回false，等待的任务会继续执行。赶时髦的人可能会用这种方式：  
```java
emails.parallelStream().forEach(this::sendEmail);
```  
叫我老古董吧，但是我仍然喜欢自己控制并发线程数。还有一种可选的关闭线程池的方法是`shutdown()`：  
```java
final List<Runnable> rejected = executorService.shutdownNow();
log.debug("Rejected tasks: {}", rejected.size());
```  
如果是这种方式，所有队列中等待的任务都会被抛弃，方法立即返回，同时，执行中的任务会继续执行。  

#### 4. 谨慎处理线程中断  
`Future`接口有个鲜为人知的方法是取消任务，再次我不赘述，参看我之前的博文[InterruptedException and interrupting threads explained](http://www.nurkiewicz.com/2014/05/interruptedexception-and-interrupting.html).  

#### 5. 监控和限制等待队列的长度  
线程池大小设置不合理会导致系统变慢、不稳定或内存溢出。如果设置的线程数太小，等待队列就会不断增长，需要消耗大量内存。反之，如果线程数配置的太大，也会导致系统变慢，因为会导致频繁的上下文切换。因此监控和限制队列长度是很重要的。如果线程池过载，我们需要临时地拒绝掉新提交的任务。  
```java
final BlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(100);
executorService = new ThreadPoolExecutor(n, n,
0L, TimeUnit.MILLISECONDS,
queue);
```  
以上代码等价于`Executors.newFixedThreadPool(n)`，但是不同于默认的无限制的`LinkedBlockingQueue`，我们使用了固定长度为100的`ArrayBlockingQueue`。这就意味着，如果当前已经有100个任务在等待队列中(n个任务正在执行)，新加入的任务会被拒绝，并抛出`RejectedExecutionException`。由于变量`queue`是外部可访问的，因此我们可以定期的调用队列的`size()`方法，然后通过日志、JMX等监控机制来做监控报警。  

#### 6. 记得处理线程的异常  
以下代码片段执行的结果是什么？  
```java
executorService.submit(() -> {
System.out.println(1 / 0);
});
```  
这里我被坑了好多次，结果是什么都不会显示出来，没有显示异常`java.lang.ArithmeticException: / by zero`，线程池把这个异常给吞掉了，就像什么都没有发生一样。如果是很早之前的代码使用`java.lang.Thread`创建线程，那么`UncaughtExceptionHandler`就会起作用了。但是如果使用线程池，那就要小心了。如果提交的任务是`Runnable`类型(像示例代码一样，不返回任何结果)，你必须把线程执行的代码整体`try-catch`住，然后至少记录下异常日志。如果提交的是`Callable`类型的任务，确保通过`get()`方法获取结果，并且`try-catch`住`get()`方法并且抛出异常。  
```java
final Future<Integer> division = executorService.submit(() -> 1 / 0);
//below will throw ExecutionException caused by ArithmeticException
division.get();
```  
有趣的是，甚至连`Spring`框架使用注解`@Async`也有这个bug，参见[SPR-8995](https://jira.spring.io/browse/SPR-8995)和[SPR-12090](https://jira.spring.io/browse/SPR-12090).  

#### 7. 监控队列中任务的等待时间  
一方面我们需要监控等待队列长度，但有时候，对于某一个任务，我们更关心从提交任务到任务实际执行之间等待了多长时间。这个时间间隔我们更希望能趋近于0(如果提交任务时有空闲的线程)，但很多时候这个时间会增加，因为需要在等待队列中缓冲一会儿。如果线程池没有设置固定的线程数，那么，此时就会创建新的线程，这个过程也需要一定的时间。为了能准确地监控到这个指标，如下，对原始的`ExecutorService`做一层封装。  
```java
public class WaitTimeMonitoringExecutorService implements ExecutorService {
private final ExecutorService target;
public WaitTimeMonitoringExecutorService(ExecutorService target) {
this.target = target;
}
@Override
public <T> Future<T> submit(Callable<T> task) {
final long startTime = System.currentTimeMillis();
return target.submit(() -> {
final long queueDuration = System.currentTimeMillis() - startTime;
log.debug("Task {} spent {}ms in queue", task, queueDuration);
return task.call();
}
);
}
@Override
public <T> Future<T> submit(Runnable task, T result) {
return submit(() -> {
task.run();
return result;
});
}
@Override
public Future<?> submit(Runnable task) {
return submit(new Callable<Void>() {
@Override
public Void call() throws Exception {
task.run();
return null;
}
});
}
//...
}
```  
这个类没有完全实现，但从以上代码，应该可以看出监控实现的原理。一个任务一旦提交，立刻开始计时，当任务开始执行时，停止计时，那么，这个时间段就是任务等待的时间。注意，不要被代码迷惑了，看起来`startTime`和`queueDuration`之间间隔这么短，时间上记录开始和结束是在不同的线程，相隔了数毫秒甚至几秒。执行结果如下：  
>Task com.nurkiewicz.MyTask@7c7f3894 spent 9883ms in queue

#### 8. 保存客户端调用堆栈信息  
