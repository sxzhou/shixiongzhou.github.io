---
layout: post
title:  "使用lambda表达式简化代码"
date:   2018-02-11 13:11:23
categories: java
tags: java
author: "sxzhou"
---  

最近阅读《Java8 in action》, Java8的一个重要的变化就是更好地支持行为参数化，这也是函数式编程范式的基础。传统命令式编程的目的是操作值，一切围绕值来展开，参数传递依也是赖值，但是Scala和Groovy等编程语言已经证明，方法参数化是正确的选择，应该让方法成为一等公民，他可以让代码更加简单，并且表达更直观丰富。Java8从语言层面支持了方法引用和lambda表达式。

lambda表达式是一个匿名函数，它基于数学中的λ演算得名，直接对应于其中的lambda抽象(lambda abstraction)，是一个匿名函数，即没有函数名的函数。
要理解lambda表达式，首先要了解的是函数式接口（functional interface）。简单来说，函数式接口是只包含一个抽象方法的接口。比如Java标准库中的java.lang.Runnable和java.util.Comparator都是典型的函数式接口。对于函数式接口，除了可以使用Java中标准的方法来创建实现对象之外，还可以使用lambda表达式来创建实现对象。这可以在很大程度上简化代码的实现。在使用lambda表达式时，只需要提供形式参数和方法体。由于函数式接口只有一个抽象方法，所以通过lambda表达式声明的方法体就肯定是这个唯一的抽象方法的实现，而且形式参数的类型可以根据方法的类型声明进行自动推断。  



具体关于函数式编程的思想，Java8对lambda表达式的支持，可以参考一下博客:  
[http://www.ruanyifeng.com/blog/2012/04/functional_programming.html](http://www.ruanyifeng.com/blog/2012/04/functional_programming.html)  
[https://www.ibm.com/developerworks/cn/java/j-ft20/index.html](https://www.ibm.com/developerworks/cn/java/j-ft20/index.html)  
[http://www.infoq.com/cn/articles/Java-se-8-lambda](http://www.infoq.com/cn/articles/Java-se-8-lambda)  
[http://www.importnew.com/16436.html](http://www.importnew.com/16436.html)  


Lambda表达式的基本语法:  
>(parameters) -> expression  
or  
(parameters) -> { statements; }

它由三部分构成
1. 参数列表
2. 符号 ->  
3. 函数体 : 有多个语句,可以用{} 包括, 如果需要返回值且只有一个语句,可以省略 return  

具体语法细节和Java的支持请参考:  
[http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/Lambda-QuickStart/index.html#overview](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/Lambda-QuickStart/index.html#overview)

来个例子对比下lambda表达式的神奇之处:  
我们要对某次考试的成绩做统计，找出其中没有及格(分数小于60)的学生，并且按照分数从高到低排序。  
定义一个类`StudentScore`，包含学生姓名和考试分数两个字段。  
```java
public class StudentScore {
    private String name;
    private Integer score;

    public StudentScore(){

    }

    public StudentScore(String name, Integer score) {
        this.name = name;
        this.score = score;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getScore() {
        return score;
    }

    public void setScore(Integer score) {
        this.score = score;
    }

    @Override
    public String toString() {
        return "StudentScore{" +
                "name='" + name + '\'' +
                ", score=" + score +
                '}';
    }
}

```
模拟一个测试的List  
```java
List<StudentScore> studentScoreList = Lists.newArrayList(new StudentScore("Tom", 80), new StudentScore
                ("Alice", 43), new StudentScore("Bob", 55), new StudentScore("Cassy", 51));
```
#### 1. 传统java7的实现
```java
        List<StudentScore> failedStudentScoreList = Lists.newArrayList();
        // filter
        for (StudentScore studentScore : studentScoreList) {
            if (studentScore.getScore().compareTo(60) < 0) {
                failedStudentScoreList.add(studentScore);
            }
        }
        // sort
        Collections.sort(failedStudentScoreList, new Comparator<StudentScore>() {
            @Override
            public int compare(StudentScore o1, StudentScore o2) {
                return o1.getScore().compareTo(o2.getScore());
            }
        });
        // transform
        List<String> failedNames = Lists.newArrayList();
        for(StudentScore studentScore : failedStudentScoreList){
            failedNames.add(studentScore.getName());
        }
        System.out.println(failedNames);
```  
#### 2. 使用guava简化代码  
使用guava提供的filter,transform等集合工具可以一定程度上简化代码，避免了for循环等代码实现细节，但是需要注意`lazy load`问题，之前的博客专门分析过。
```java
List<StudentScore> failedStudentList = Lists.newArrayList(Iterables.filter(studentScoreList, new
        Predicate<StudentScore>() {
            @Override
            public boolean apply(StudentScore input) {
                return input.getScore().compareTo(60) < 0;
            }
        }));
Collections.sort(failedStudentList, new Comparator<StudentScore>() {
    @Override
    public int compare(StudentScore o1, StudentScore o2) {
        return o1.getScore().compareTo(o2.getScore());
    }
});
List<String> failedNames = Lists.transform(failedStudentList, new Function<StudentScore, String>() {
    @Override
    public String apply(StudentScore input) {
        return input.getName();
    }
});
System.out.println(failedNames);
```  
#### 3. lambda表达式结合guava  
使用lambda表达式，可以极大地简化代码，原有的**函数式接口**(只定义一个抽象方法的接口)可以完全用lambda表达式替换，省去了匿名接口的一系列模板代码。
```java
List<StudentScore> failedStudentList = Lists.newArrayList(Iterables.filter(studentScoreList, studentScore -> studentScore.getScore() < 60));
Collections.sort(failedStudentList, Comparator.comparing(StudentScore::getScore));
List<String> failedNames = Lists.transform(failedStudentList, StudentScore::getName);
System.out.println(failedNames);
```  
其实，Java8新增了`Stream API`对于处理这种集合操作更合适，不仅代码更加简洁(以上功能一行代码实现)，逻辑更易读，还自带并行处理能力，下回再整理。



