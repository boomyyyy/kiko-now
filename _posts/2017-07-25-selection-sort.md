---
layout: post
title: "选择排序"
description: "java 实现选择排序"
date: 2017-07-25
tags: [Java, 算法]
comments: true
share: true
---

> 选择排序基本思想如下，在一个数组中，从一个元素开始寻找到最小的元素，确定最小元素下标后与第一个元素下标互换，然后从第二个元素开始确定最小元素的下标并与第二个元素互换，以此类推。时间复杂度为 O(n^2)

```java
public class SelectionSort {

  private SelectionSort() {

  }

  public static void sort(int[] arr){
    int length = arr.length;
    for (int i = 0; i < length; i ++) {
      int minIdx = i;
      for (int j = i; j < length; j++) {
        if (arr[i] > arr[j]) {
          minIdx = j;
        }
      }
      swap(arr, i, minIdx);
    }
  }

  public static void swap(int[] arr, int i, int j) {
    int t = arr[i];
    arr[i] = arr[j];
    arr[j] = t;
  }

  public static void main(String[] args) {
    int[] arr = {7,6,5,3,10,4,9,8,2,1};
    sort(arr);
    for (int i = 0; i < arr.length; i ++) {
      System.out.print(arr[i] + " ");
    }
  }

}
```
