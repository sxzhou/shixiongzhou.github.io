---
layout: post
title:  "guava中Lists.transform()执行过程分析"
date:   2017-12-13 09:09:22
categories: java
tags: java
author: "sxzhou"
---

今天碰到一个问题，看似正确的代码，遍历一个`List`,给每个`item`中的某个属性赋值，再次遍历获取这个属性，竟然没有变化，之前的赋值操作并没有生效，猛然想起之前使用`guava`的集合框架碰到的相似的问题，因为`guava`集合框架很多使用`lazy apply`和`view`，因此表面上无懈可击的代码，真正的执行过程并不是我们预想的。　　
```java
public class App {

    public static void main(String[] args) {
        // step1:构造原始list
        List<A> aList = Lists.newArrayList();
        A a = new A();
        a.setA("original");
        aList.add(a);
        for (A item : aList) {
            System.out.println("original A:" + item.getA());
        }
        // step2:使用transform方法遍历aList,将A转换成Ｂ
        List<B> transformed = Lists.transform(aList, new Function<A, B>() {
            public B apply(A input) {
                B b = new B();
                b.setB("transformed");
                return b;
            }
        });
        // step3:遍历经过transform的List<B>，将每个item的b字段赋值为afterSet
        for (B item : transformed) {
            System.out.println("guava list transformed B:" + item.getB());
            item.setB("afterSet");
        }
        for (B item : transformed) {
            System.out.println("after set B:" + item.getB());
        }
    }

    static class A {
        private String a;

        public A() {
        }

        public String getA() {
            return a;
        }

        public void setA(String a) {
            this.a = a;
        }
    }

    static class B {
        private String b;

        public B() {
        }

        public String getB() {
            return b;
        }

        public void setB(String b) {
            this.b = b;
        }
    }
}
```  
执行结果:
>original A:original  
guava list transformed B:transformed  
after set B:transformed  

可见，`step3`并未生效，或者说没有按照我们预想的方式生效。　　
>Returns a list that applies {@code function} to each element of {@code
 fromList}. The returned list is a transformed view of {@code fromList};
 changes to {@code fromList} will be reflected in the returned list and vice versa.  
 Since functions are not reversible, the transform is one-way and new items cannot be stored in the returned list. The {@code add},{@code addAll} and {@code set} methods are unsupported in the returned list.  
 The function is applied lazily, invoked when needed. This is necessary for the returned list to be a view, but it means that the function will be applied many times for bulk operations like {@link List#contains} and {@link List#hashCode}. For this to perform well, {@code function} should be fast. To avoid lazy evaluation when the returned list doesn't need to be a view, copy the returned list into a new list of your choosing.    
 
 根据文档的说明，需要知道，`transform`方法返回的是一个视图，并不是一个新的列表实例，这个方法是惰性执行的，因此在`step2`并没有执行`transform`方法，而是在后续遍历通过迭代器取值时才会执行。　　
 分析源码：　　
 ```java
 public static <F, T> List<T> transform(
      List<F> fromList, Function<? super F, ? extends T> function) {
    return (fromList instanceof RandomAccess)
        ? new TransformingRandomAccessList<F, T>(fromList, function)
        : new TransformingSequentialList<F, T>(fromList, function);
  }
 ```  
 `ArrayList`实现了`RandomAccess`，因此`transform`返回的结果类型是`TransformingRandomAccessList`.  
 ```java
 private static class TransformingRandomAccessList<F, T> extends AbstractList<T>
      implements RandomAccess, Serializable {
    final List<F> fromList;
    final Function<? super F, ? extends T> function;

    TransformingRandomAccessList(List<F> fromList, Function<? super F, ? extends T> function) {
      this.fromList = checkNotNull(fromList);
      this.function = checkNotNull(function);
    }
 ```  
 `TransformingRandomAccessList`包含两个属性，原始列表`fromList`，转换的方法`function`，构造这个对象的时候并没有调用`transform`方法`new`一个`List`，当调用`get`方法时，才会执行转换。如果使用迭代器遍历，会发生什么呢？　　
 ```java
 @Override
    public ListIterator<T> listIterator(int index) {
      return new TransformedListIterator<F, T>(fromList.listIterator(index)) {
        @Override
        T transform(F from) {
          return function.apply(from);
        }
      };
    }
 ```  
 返回的是`TransformedListIterator`，提供了原始列表的迭代器，并实现了`transform`方法，遍历取值时，执行的是：
 ```java
 @Override
  public final T next() {
    return transform(backingIterator.next());
  }
 ```  
 也就是，先使用原始列表的迭代器，然后执行`transform`方法，这就是`lazy apply`的实现过程，