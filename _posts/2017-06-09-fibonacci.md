---
layout: post
title: "Java 实现斐波那契数列"
description: "Java 代码示例"
date: 2017-06-09
tags: [算法,Java]
comments: true
share: true
---

```java
public class Fibonacci {
  public static void main(String[] args) {
      System.out.println(fibonacci1(9));
      System.out.println(fibonacci2(9));
  }
  // 递归实现
  public static int fibonacci1(int n) {
    if (n <= 2) {
      return 1;
    } else {
      return fibonacci1(n-1) + fibonacci1(n - 2);
    }
  }

  // 递推实现
  public static int fibonacci2(int n) {
    if (n <= 2) {
      return 1;
    } else {
      int n1 = 1, n2 = 1, sn = 0;
      for (int i = 0; i < n-2; i++) {
        sn = n1 + n2;
        n1 = n2;
        n2 = sn;
      }
      return sn;
    }
  }
}
```
