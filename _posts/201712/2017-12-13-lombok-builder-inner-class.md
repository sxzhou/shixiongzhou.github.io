---
layout: post
title:  "non-static inner class使用lombok注解@Builder编译失败分析"
date:   2017-12-13 09:09:22
categories: java
tags: java
author: "sxzhou"
---

今天编译一个工程，报错`error: modifier static not allowed here`，这种错误，编码过程中
IDEA就应该发现的，但本地并没有报错。diff代码发现，本次新增了一些`lombok`的注解简化代码。单独编译修改的
文件，果然报错，对于非`inner class`，测试一下`@Builder`注解:
```java
@Builder
public class LombokBuildTest {
    private String a;
    private int b;
}
```
反编译，看看`lombok`帮我们生成了什么代码:
```java
public class LombokBuildTest {
    private String a;
    private int b;

    LombokBuildTest(String a, int b) {
        this.a = a;
        this.b = b;
    }

    public static LombokBuildTest.LombokBuildTestBuilder builder() {
        return new LombokBuildTest.LombokBuildTestBuilder();
    }

    public static class LombokBuildTestBuilder {
        private String a;
        private int b;

        LombokBuildTestBuilder() {
        }

        public LombokBuildTest.LombokBuildTestBuilder a(String a) {
            this.a = a;
            return this;
        }

        public LombokBuildTest.LombokBuildTestBuilder b(int b) {
            this.b = b;
            return this;
        }

        public LombokBuildTest build() {
            return new LombokBuildTest(this.a, this.b);
        }

        public String toString() {
            return "LombokBuildTest.LombokBuildTestBuilder(a=" + this.a + ", b=" + this.b + ")";
        }
    }
}
```
正如我们熟悉的Builder，内部会定义一个`static`类和供外部访问的`static`入口方法，根据[java语言规范](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html)中对于`inner class`的规定:
>It is a compile-time error if an inner class declares a static initializer (§8.7).  
It is a compile-time error if an inner class declares a member that is explicitly or implicitly static, unless the member is a constant variable (§4.12.4).

因此，如果使用了`lombok`的`@Builder`注解，编译时插入的代码中包含了static方法，如果在非静态内部类上使用，会导致编译错误，因此需要将内部类声明为静态，修改代码，加上`static`修饰符：
```java
public class LombokTest {

    @Builder
    private static class InnerClass{
        private String a;
        private int b;
    }
}
```
实际上已经有人给`lombok`作者反馈过这个问题.  
[@Builder on non-static inner classes does not work. #1302](https://github.com/rzwitserloot/lombok/issues/1302)  
作者回复:  
>Tests case coming up for more detail on this.  
Probably correct solution:  
Generate the builder method and the builderclass as a sibling to the inner; there does not appear to be anything else that's workable.  
For now you just get an error that 'static' is not allowed here; that'll do fine until we get around to solving this properly.

按照目前通用的Builder实现方式，确实没有什么好的方法来处理这个问题。因此，使用这个方法一定要注意。   
之前使用`lombok`时关于默认构造方法也踩过一次坑，后续再整理出来。   
总之，lombok虽然好用，代码看起来美观不少，但一定要反编译出来看看，到底它帮我们添加了哪些代码，是不是和我们预想的一样。　　
## 扩展一下　　
之前一直以为lombok是使用字节码操作实现的，实际上是通过[JSR 269 Pluggable Annotation Processing API](http://www.oracle.com/splash/jcp.org/maintenance/index.html)机制实现的，也就是lombok实现了JSR 269 Pluggable Annotation Processing API，编译阶段，javac会调用这个api，修改生成的抽象语法树，在这一步插入了新特性，之后生成字节码。  
**参考**  
[https://www.ibm.com/developerworks/cn/java/j-lombok/index.html](https://www.ibm.com/developerworks/cn/java/j-lombok/index.html)  
[http://blog.csdn.net/dslztx/article/details/46715803](http://blog.csdn.net/dslztx/article/details/46715803)  
[http://blog.csdn.net/zhuhai__yizhi/article/details/49999151](http://blog.csdn.net/zhuhai__yizhi/article/details/49999151)  
[http://www.infoq.com/cn/articles/Living-Matrix-Bytecode-Manipulation](http://www.infoq.com/cn/articles/Living-Matrix-Bytecode-Manipulation)