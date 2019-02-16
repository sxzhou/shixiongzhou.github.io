---
layout: post
title:  "guava中topK算法实现解析"
date:   2018-03-11 11:00:51
categories: article
tags: algorithm guava
author: "sxzhou"
---  
对于topK问题，我们首先想到的是通过最小(大)堆实现，jdk也提供了相关的实现`PriorityQueue`，可以很方便地实现，建堆时间复杂度O(n)，实现原理不赘述，同时guava也提供了相关的工具，在`Ordering`中提供了`leastOf`和对应的`greatestof`方法，获取集合中的最小或最大的k个元素，实现原理和优先队列不一样。可以看到guava的设计者更注重工程实践的需要。  
```java
public <E extends T> List<E> leastOf(Iterator<E> elements, int k) {
   checkNotNull(elements);
   checkNonnegative(k, "k");

   if (k == 0 || !elements.hasNext()) {
     return ImmutableList.of();
   } else if (k >= Integer.MAX_VALUE / 2) {
     // k is really large; just do a straightforward sorted-copy-and-sublist
     ArrayList<E> list = Lists.newArrayList(elements);
     Collections.sort(list, this);
     if (list.size() > k) {
       list.subList(k, list.size()).clear();
     }
     list.trimToSize();
     return Collections.unmodifiableList(list);
   }
   // and then ......
 }
```  
可以看到，有一些防御性的判断，如果`k >= Integer.MAX_VALUE / 2`，直接排序然后取子序列，因为下面的算法需要分配`2*k`的数组，如果`k`太大，还不如直接排序。  
```java
    int bufferCap = k * 2;
    @SuppressWarnings("unchecked") // we'll only put E's in
    E[] buffer = (E[]) new Object[bufferCap];
    E threshold = elements.next();
    buffer[0] = threshold;
    int bufferSize = 1;
    // threshold is the kth smallest element seen so far.  Once bufferSize >= k,
    // anything larger than threshold can be ignored immediately.

    while (bufferSize < k && elements.hasNext()) {
      E e = elements.next();
      buffer[bufferSize++] = e;
      threshold = max(threshold, e);
    }
```   
然后分配`2*k`长度的数组，先填充`k`个元素，`threshold`记录了当前最小的`k`个元素的上限。  
```java
      E e = elements.next();
      if (compare(e, threshold) >= 0) {
        continue;
      }

      buffer[bufferSize++] = e;
```
从`k+1`个元素开始，根据`threshold`的定义可以知道，如果大于`threshold`，肯定不是topK的，直接抛弃。  
```java
if (bufferSize == bufferCap) {
        // We apply the quickselect algorithm to partition about the median,
        // and then ignore the last k elements.
        int left = 0;
        int right = bufferCap - 1;

        int minThresholdPosition = 0;
        // The leftmost position at which the greatest of the k lower elements
        // -- the new value of threshold -- might be found.

        while (left < right) {
          int pivotIndex = (left + right + 1) >>> 1;
          int pivotNewIndex = partition(buffer, left, right, pivotIndex);
          if (pivotNewIndex > k) {
            right = pivotNewIndex - 1;
          } else if (pivotNewIndex < k) {
            left = Math.max(pivotNewIndex, left + 1);
            minThresholdPosition = pivotNewIndex;
          } else {
            break;
          }
        }
        bufferSize = k;

        threshold = buffer[minThresholdPosition];
        for (int i = minThresholdPosition + 1; i < bufferSize; i++) {
          threshold = max(threshold, buffer[i]);
        }
      }
```  
当填满`2*k`个元素后，就通过快速选择算法找到中位数`k`的值,那么左边就是当前元素序列的最小`topK`，右边舍弃，循环遍历余下的序列，最终就会得到原始输入的所有元素的最小的`k`个元素。  
guava提供的算法，实现了时间复杂度O(n)和空间复杂度O(k)，不需要把所有的元素都载入数组，相对于`PriorityQueue`，实现过程复杂一些，但使用更小的空间，这对于工程实践是很重要的。  


