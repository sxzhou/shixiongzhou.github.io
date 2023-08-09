---
layout: post
title:  "从码农到工匠-设计原则与设计模式"
date:   2023-03-12 22:13:00
categories: article
tags: go
author: "sxzhou"
---   

对每个软件开发者来说，`SOLID`软件设计原则以及GoF总计的23种设计模式是耳熟能详的，其实完整地记下这些内容实际是很困难的，
更多的是在开发中能深入理解，在做技术设计时能灵活使用。   
这些概念已经提出很久了，其实除了这些条款，在开发大型企业级应用时，开发者们也总结了一些其他的设计原则和设计模式，在此搜集总结一下。
---
## 1. SOLID设计原则
`SOLID`的定义：
* Single Responsibility Principle（SRP）：单一职责原则
* Open Closed Principle（OCP）：开闭原则
* Liskov Substitution Principle（LSP）：里氏替换原则
* Interface Segregation Principle（ISP）：接口隔离原则
* Dependence Inversion Principle（DIP）：依赖倒置原则   

对于这些原则已经有很多文章专门解读，在此不做展开，这些原则并不是孤立的，一个可以是其他的加强或基础，通常在设计中
违反一个原则可能同时也违反了多个。  
需要注意的是，开闭原则和里氏代换原则是设计目标，单一职责原则、接口分离原则以及依赖倒置原则是设计方法。  
还有一点需要强调，依赖倒置原则是在工作中经常用到的，而且也是常常被忽略的，需要开发者重视，**上层只能依赖下层的抽象**。DIP是
一种编程思想，**面向接口编程**则是实现它的一种技法。遵循DIP会大大提升系统灵活性，在构建大型软件系统时要格外重视。  
---
## 2. 其他重要的设计原则
除了`SOLID`设计原则，还有很多非常有用的总结，都是前人在开发实践中的提炼，需要体会消化和应用。 

**DRY**   
`DRY`即`Don't Repeat Yourself`，通俗来讲就是不要有重复的代码和模块，如果出现，需要考虑抽象出来成为通用解决方法，否则会降低
代码的灵活和简洁，同时可能导致代码之间的矛盾。   
**YAGNI**   
`YAGNI`即`You Ain't Gonna Need It`。意思是你不会需要它，这个原则也是很多有开发经验的开发者容易犯的错误，往往把一个功能过度设计，
考虑的过于长远，实际后来可能根本用不上，浪费了资源也增加了系统复杂度。  
**Rule of Three**   
可以看到，`DRY`和`YAGNI`是存在矛盾的，一个是希望不要出现重复，尽可能抽象复用，另一个是反对过度设计而导致浪费。因此提出了这个折中
的`三次原则`，也就是当碰到一个功能出现第三次复用时，我们才把它抽象出来，实践中，需要我们自己把握好这个度。   
**KISS**   
`KISS`即`Keep It Simple and Stupid`，把简单的事情变复杂是很容易的，反之则很困难。功能实现过于复杂往往是因为前期没有思考清楚，人为的
把简单功能复杂化了。简单清晰的实现往往是更好的选择，既便于维护也便于其他人理解。   
**POLA**   
`POLA`即`Principle of least astonishment`，这是从代码复杂度考虑，写代码不是写小说，不需要高潮惊喜反转之类的技巧，简洁清晰的实现，包括
代码之外的注释命名都要考虑这个原则。   

看了以上这些原则，有一个总体的规范就是代码需要设计但反对过度设计，清晰简单的实现往往要好于奇技淫巧，在我所接触过得大型软件项目中，可以深刻
体会到这个好处。   

---
## 3. 设计模式   
设计原则是一种编程思想或者是软件设计的艺术，是一种指导原则，而设计模式则是经验落地，利用模式，我们可以让一个解决
方案重复使用，而不是重复造轮子。   
`GoF`的23种设计模式有很多书籍文章专门解读，一个软件系统中，设计模式不是越多越好，这样反而有炫技之嫌，提升了系统复杂度，
违反了KISS原则，合适的场景有合适的理由运用合适的模式，才是考验设计能力的核心所在。   
在这23中设计模式之外，也有很多开发者总结补充了其他模式，特别是在应对复杂业务系统流程编排场景，以下这几个值得参考借鉴。  

**拦截器模式**   
很多场景下，我们需要在业务流程前后提供一些通用的业务无关的操作，比如日志记录、性能统计、安全控制等，如果是简单的实现，必然导致很多重复代码，这时就需要
用到拦截器模式，它和面向切面编程思想是相似的，相对于使用常用的代理技术实现的切面编程，拦截器模式更加灵活，命名可以更好的表达前后置处理的含义，添加和删除
更加灵活，从代码阅读上也更加简单明了。   
[![pPeGj2Q.png](https://s1.ax1x.com/2023/08/09/pPeGj2Q.png)](https://imgse.com/i/pPeGj2Q)
在拦截器模式中，主要包含以下角色: 
* TargetInvocation：包含了一组Interceptor和一个Target对象，确保在Target处理请求前后，按照定义顺序调用Interceptor做前置和后置处理。  
* Target：处理请求的目标接口。  
* TargetImpl：实现了Target接口的对象。
* Interceptor：拦截器接口。
* InterceptorImpl：拦截器实现，用来在Target处理请求前后做切面处理。
```go
public interface Target{
    public Response execute(Request request);
}

public interface Interceptor {
    public Response intercept(TargetInvocation targetInvocation);
}

public class TargetInvocation {
    private List<Interceptor> interceptorList = new ArrayList<>();
    private Iterator<Interceptor> interceptors;
    private Target target;
    private Request request;
    public Response invoke(){
        if( interceptors.hasNext() ){
            Interceptor interceptor = interceptors.next();
            interceptor.intercept(this);
        }
        return target.execute(request);
    }
	
    public void addInterceptor(Interceptor interceptor){
        interceptorList.add(interceptor);
        interceptors = interceptorList.iterator();
    }
}

public class AuditInterceptor implements Interceptor{
    @Override
    public Response intercept(TargetInvocation targetInvocation) {
        if(targetInvocation.getTarget() == null) {
            throw new IllegalArgumentException("Target is null");
        }
        System.out.println("Audit Succeeded ");
        return targetInvocation.invoke();
    }
}

public class LogInterceptor implements Interceptor {
    @Override
    public Response intercept(TargetInvocation targetInvocation) {
        System.out.println("Logging Begin");
        Response response = targetInvocation.invoke();
        System.out.println("Logging End");
        return response;
    }
}

public class InterceptorDemo {
    public static void main(String[] args) {
        TargetInvocation targetInvocation = new TargetInvocation();
        targetInvocation.addInterceptor(new LogInterceptor());
        targetInvocation.addInterceptor(new AuditInterceptor());
        targetInvocation.setRequest(new Request());
        targetInvocation.setTarget(request->{return new Response();});
        targetInvocation.invoke();
    }
}
```

**插件模式**   
在企业级软件系统中，扩展性是一个很重要的考虑点，设计任何功能都要考虑到未来的变化，通常简单的场景下，我们可以考虑使用策略模式，这也是在实际生产
中使用较多的一类设计模式，但是当需要添加一个新的策略时，我们就得重新实现代码，然后编译打包部署，这个流程同样繁琐，且对原有流程有侵入，如果我们把
变化的部分变成扩展能力，作为一个独立的组件，甚至可以热部署，并且在运行时实现插件的加载和卸载，这样才是真正的即插即用。   
因此插件模式更多的是一种软件架构的思想，并不是简单的几行代码实现，而是要实现一套完整的插件框架。  
以下是一些关于插件化的思考及实现：  
[插件式可扩展架构设计心得](https://zhuanlan.zhihu.com/p/372381276)  
[Java Plug-in Framework (JPF) Project](https://jpf.sourceforge.net/about.html)
