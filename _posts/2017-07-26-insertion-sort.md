---
layout: post
title: "插入排序"
description: "java 实现插入排序"
date: 2017-07-26
tags: [Java, 算法]
comments: true
share: true
---

> 插入排序的思想有点像打扑克抓牌的时候，我们插入扑克牌的做法。想象一下，抓牌时，我们都是把抓到的牌按顺序放在手中。因此每抓一张新牌，我们都将其插入到已有的排好序的手牌当中，注意体会刚才的那句话。也就是说，插入排序的思想是，将新来的元素按顺序放入一个已有的有序序列当中。

```java
public class InsertionSort {

  private InsertionSort() {

  }

  public static void sort(int arr) {
    int length = arr.length;
    for (int i = 0; i < length; i++) {
      int num = arr[i];
      int j = i;
      for (; j > 0 && arr[j-1] > num; j --) {
        arr[i] = arr[j-1];
      }
      arr[j] = num;
    }
  }

}
```
