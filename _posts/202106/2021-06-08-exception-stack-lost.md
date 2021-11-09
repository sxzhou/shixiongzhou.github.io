---
layout: post
title:  "Java日志异常栈丢失的原因"
date:   2021-06-08 23:27:00
categories: article
tags: java
author: "sxzhou"
---   

最近在排查线上问题时碰到个诡异的问题，error日志中的异常栈没有了，只有一行：
```java
java.lang.NullPointerException
```
于是去查了网上文章，都指向一个JVM启动参数：
`OmitStackTraceInFastThrow`   
这个参数默认是开启的，解释是：
>The compiler in the server VM now provides correct stack backtraces for all "cold" built-in exceptions. For performance purposes, when such an exception is thrown a few times, the method may be recompiled. After recompilation, the compiler may choose a faster tactic using preallocated exceptions that do not provide a stack trace. To disable completely the use of preallocated exceptions, use this new flag: -XX:-OmitStackTraceInFastThrow.

JVM对一些特定的异常类型做了Fast Throw优化，如果检测到在代码里某个位置连续多次抛出同一类型异常的话，C2会决定用Fast Throw方式来抛出异常，而异常Trace即详细的异常栈信息会被清空。这种异常抛出速度非常快，因为不需要在堆里分配内存，也不需要构造完整的异常栈信息。  
做个小测试：
```java
public class StackLostTest {
    public static void main(String[] args) {
        for (int i = 0; i < 200000; i++) {
            try {
                Integer a = null;
                Integer b = a + 1;
            } catch (Exception e) {
                int length = e.getStackTrace().length;
                if (length < 1) {
                    System.out.println("stack lost! i=" + i);
                    e.printStackTrace();
                    break;
                }
            }
        }
    }
}
```
在我本地的运行结果是：
```java
stack lost! i=115715
java.lang.NullPointerException
```
这个抛NPE的的代码在执行了115715次后被优化了，我们可以设置启动参数`-XX:-OmitStackTraceInFastThrow`，这样可以使得异常栈完整展示，但是在
生产环境，这种方式是不好的，可能导致`日志风暴`，我们应该找到最开始抛异常的时间点，查看当时的日志，所以说`日志持久化`和`异常及时响应`才是正解。

---
**更深入些**
执行这个优化的代码如下：
```java
  // If this throw happens frequently, an uncommon trap might cause
  // a performance pothole.  If there is a local exception handler,
  // and if this particular bytecode appears to be deoptimizing often,
  // let us handle the throw inline, with a preconstructed instance.
  // Note:   If the deopt count has blown up, the uncommon trap
  // runtime is going to flush this nmethod, not matter what.
  if (treat_throw_as_hot
      && (!StackTraceInThrowable || OmitStackTraceInFastThrow)) {
    // If the throw is local, we use a pre-existing instance and
    // punt on the backtrace.  This would lead to a missing backtrace
    // (a repeat of 4292742) if the backtrace object is ever asked
    // for its backtrace.
    // Fixing this remaining case of 4292742 requires some flavor of
    // escape analysis.  Leave that for the future.
    ciInstance* ex_obj = NULL;
    switch (reason) {
    case Deoptimization::Reason_null_check:
      ex_obj = env()->NullPointerException_instance();
      break;
    case Deoptimization::Reason_div0_check:
      ex_obj = env()->ArithmeticException_instance();
      break;
    case Deoptimization::Reason_range_check:
      ex_obj = env()->ArrayIndexOutOfBoundsException_instance();
      break;
    case Deoptimization::Reason_class_check:
      if (java_bc() == Bytecodes::_aastore) {
        ex_obj = env()->ArrayStoreException_instance();
      } else {
        ex_obj = env()->ClassCastException_instance();
      }
      break;
    }
    ... ...
}
```
可以看到，触发优化的判断条件是：
`treat_throw_as_hot && (!StackTraceInThrowable || OmitStackTraceInFastThrow)`，即：
* 这个代码位置持续抛出异常
* StackTraceInThrowable=false或者OmitStackTraceInFastThrow=true
* 命中了如上几种异常

是否能在运行时临时关闭这个优化，让开发者能快速定位问题，这里有些讨论：  
[https://bugs.openjdk.java.net/browse/JDK-8046503](https://bugs.openjdk.java.net/browse/JDK-8046503)  

其实上面这段测试代码，在本地运行时会有个奇怪的现象，异常栈丢失后再执行一段时间后有出现了完整异常栈了，这里也有人复现过：  
[https://bugs.openjdk.java.net/browse/JDK-8073432](https://bugs.openjdk.java.net/browse/JDK-8073432) 
推测是JIT编译优化和反优化造成的，需要更多研究。