---
layout: post
title:  "用Lambda重构OOP设计模式"
date:   2019-11-03 19:02:00
categories: reading
tags: java
author: "sxzhou"
---  
*《Java8实战》*

#### 策略模式   
策略模式代表了解决一类算法的通用解决方案，你可以在运行时选择使用哪种方案。包含三部分内容：   
* 一个代表某个算法的接口
* 一个或多个该接口的具体实现，它们代表了算法的多种实现
* 一个或多个使用策略对象的客户  

一个验证文本输入的例子：   
策略接口：  
```java 
interface ValidationStrategy {
    boolean execute(String s);
}
```  
算法实现：   
```java
static class IsAllLowerCase implements ValidationStrategy {

    @Override
    public boolean execute(String s) {
        return s.matches("[a-z]+");
    }
}

static class IsNumeric implements ValidationStrategy {

    @Override
    public boolean execute(String s) {
        return s.matches("\\d+");
    }
}
```
客户端使用：  
```java
private static class Validator {
    private final ValidationStrategy validationStrategy;

    public Validator(ValidationStrategy validationStrategy) {
        this.validationStrategy = validationStrategy;
    }

    public boolean validate(String s) {
        return validationStrategy.execute(s);
    }
}

Validator v1 = new Validator(new IsNumeric());
// false
System.out.println(v1.validate("aaaa"));
Validator v2 = new Validator(new IsAllLowerCase());
// true
System.out.println(v2.validate("bbbb"));
```  

Lambda实现：  
```java 
Validator v3 = new Validator((String s) -> s.matches("\\d+"));
System.out.println(v3.validate("aaaa"));
Validator v4 = new Validator((String s) -> s.matches("[a-z]+"));
System.out.println(v4.validate("bbbb"));
```   

#### 模板方法   
采用某个算法框架，同时有希望有一定的灵活性，能对它的某些部分进行改进。   
例子：假设你需要编写一个简单的在线银行应用。通常，用户需要输入一个用户账户，之后应用才能从银行的数据库中得到用户的详细信息，最终完成一些让用户满意的操作。不同分行的在线银行应用让客户满意的方式可能还略有不同，比如给客户的账户发放红利，或者仅仅是少发送一些推广文件。

```java
public abstract class AbstractOnlineBanking {
    public void processCustomer(int id) {
        Customer customer = Database.getCustomerWithId(id);
        makeCustomerHappy(customer);
    }

    /**
     * 让客户满意
     *
     * @param customer
     */
    abstract void makeCustomerHappy(Customer customer);

    private static class Customer {}

    private static class Database {
        static Customer getCustomerWithId(int id) {
            return new Customer();
        }
    }
}
```    

Lambda方式：  
```java
public void processCustomer(int id, Consumer<Customer> makeCustomerHappy) {
    Customer customer = Database.getCustomerWithId(id);
    makeCustomerHappy.accept(customer);
}

public static void main(String[] args) {
        new AbstractOnlineBankingLambda().processCustomer(1337, (
            AbstractOnlineBankingLambda.Customer c) -> System.out.println("Hello!"));
}
```

#### 观察者模式  

某些事件发生时（比如状态转变），如果一个对象（通常我们称之为主题）需要自动地通知其他多个对象（称为观察者）。   
例子：你需要为Twitter这样的应用设计并实现一个定制化的通知系统。想法很简单：好几家报纸机构，比如《纽约时报》《卫报》以及《世界报》都订阅了新闻，他们希望当接收的新闻中包含他们感兴趣的关键字时，能得到特别通知。   

首先，你需要一个观察者接口，它将不同的观察者聚合在一起。它仅有一个名为 notify 的方法，一旦接收到一条新的新闻，该方法就会被调用：
```java
interface Observer{
    void inform(String tweet);
}
```  
声明不同的观察者（比如，这里是三家不同的报纸机构），依据新闻中不同的关键字分别定义不同的行为：  
```java
private static class NYTimes implements Observer {

    @Override
    public void inform(String tweet) {
        if (tweet != null && tweet.contains("money")) {
            System.out.println("Breaking news in NY!" + tweet);
        }
    }
}

private static class Guardian implements Observer {

    @Override
    public void inform(String tweet) {
        if (tweet != null && tweet.contains("queen")) {
            System.out.println("Yet another news in London... " + tweet);
        }
    }
}

private static class LeMonde implements Observer {

    @Override
    public void inform(String tweet) {
        if(tweet != null && tweet.contains("wine")){
            System.out.println("Today cheese, wine and news! " + tweet);
        }
    }
}
```
Subject为它定义一个接口：
```java
interface Subject {
    void registerObserver(Observer o);

    void notifyObserver(String tweet);
}
```    
实现 Feed 类：
```java
private static class Feed implements Subject {
    private final List<Observer> observers = new ArrayList<>();

    @Override
    public void registerObserver(Observer o) {
        observers.add(o);
    }

    @Override
    public void notifyObserver(String tweet) {
        observers.forEach(o -> o.inform(tweet));
    }
}
```    

使用Lambda:
```java
Feed feedLambda = new Feed();
feedLambda.registerObserver((String tweet) -> {
    if (tweet != null && tweet.contains("money")) {
        System.out.println("Breaking news in NY!" + tweet);
    }
});

feedLambda.registerObserver((String tweet) -> {
    if (tweet != null && tweet.contains("queen")) {
        System.out.println("Yet another news in London... " + tweet);
    }
});

feedLambda.notifyObserver("Money money money, give me money!");
```  

我们前文介绍的例子中，Lambda适配得很好，那是因为需要执行的动作都很简单，因此才能很方便地消除僵化代码。但是，观察者的逻辑有可能十分复杂，它们可能还持有状态，抑或定义了多个方法，诸如此类。在这些情形下，你还是应该继续使用类的方式。   


#### 责任链模式   

责任链模式是一种创建处理对象序列（比如操作序列）的通用方案。一个处理对象可能需要在完成一些工作之后，将结果传递给另一个对象，这个对象接着做一些工作，再转交给下一个处理对象，以此类推。   

```java
private static abstract class AbstractProcessingObject<T> {
    protected AbstractProcessingObject<T> successor;

    public void setSuccessor(AbstractProcessingObject<T> successor) {
        this.successor = successor;
    }

    public T handle(T input) {
        T r = handleWork(input);
        if (successor != null) {
            return successor.handle(r);
        }
        return r;
    }

    protected abstract T handleWork(T input);
}
```
创建两个处理对象，它们的功能是进行一些文本处理工作。  
```java
private static class HeaderTextProcessing extends AbstractProcessingObject<String> {

    @Override
    protected String handleWork(String text) {
        return "From Raoul, Mario and Alan: " + text;
    }
}

private static class SpellCheckerProcessing extends AbstractProcessingObject<String> {

    @Override
    protected String handleWork(String text) {
        return text.replaceAll("labda", "lambda");
    }
}
```    
使用：
```java
AbstractProcessingObject<String> p1 = new HeaderTextProcessing();
AbstractProcessingObject<String> p2 = new SpellCheckerProcessing();
p1.setSuccessor(p2);
String result = p1.handle("Aren't labdas really sexy?!!");
System.out.println(result);
```   
使用Lambda:  
```java
UnaryOperator<String> headerProcessing =
            (String text) -> "From Raoul, Mario and Alan: " + text;

UnaryOperator<String> spellCheckerProcessing =
        (String text) -> text.replaceAll("labda", "lambda");

Function<String, String> pipeline = headerProcessing.andThen(spellCheckerProcessing);

String result2 = pipeline.apply("Aren't labdas really sexy?!!");
System.out.println(result2);
```   

#### 工厂模式
使用工厂模式，你无需向客户暴露实例化的逻辑就能完成对象的创建。  

例子：假定你为一家银行工作，他们需要一种方式创建不同的金融产品：贷款、期权、股票，等等。  

```java
private interface Product {
}

private static class ProductFactory {
    public static Product createProduct(String name) {
        switch (name) {
            case "loan":
                return new Loan();
            case "stock":
                return new Stock();
            case "bond":
                return new Bond();
            default:
                throw new RuntimeException("No such product " + name);
        }
    }
}

static private class Loan implements Product {
}

static private class Stock implements Product {
}

static private class Bond implements Product {
}
```  
使用：
```java
Product p1 = ProductFactory.createProduct("loan");
```  

使用Lambda:  
```java
final static private Map<String, Supplier<Product>> map = new HashMap<>();

static {
    map.put("loan", Loan::new);
    map.put("stock", Stock::new);
    map.put("bond", Bond::new);
}
public static Product createProductLambda(String name) {
    Supplier<Product> p = map.get(name);
    if (p != null) {
        return p.get();
    }
    throw new RuntimeException("No such product " + name);
}
```