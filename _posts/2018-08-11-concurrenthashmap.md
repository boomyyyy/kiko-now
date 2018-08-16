---
layout: post
title: "ConcurrentHashMap 简述(JDK 1.8)"
date: 2018-08-04
tags: [Java,集合,并发]
comments: true
share: true
---

## 概述
之前了解的相关 HashMap 都是线程不安全的，今天来看下线程安全的相关 Map，其中较为熟悉的 Hashtable 和 ConcurrentHashMap 两个。其中 Hashtable 是在 HashMap 的基础上将所有的方法是 synchronized 关键字修饰，锁的作用范围是当前对象，每次只允许一个线程访问当前 map，性能极其差，而 ConcurrentHashMap 首先将存储数据的链表数组使用 volatile 关键字修饰，保证内存可见性，在查询的时候不需要加锁，极大的提升了查询性能，插入数据时缩小了锁的作用范围，先通过插入数据的 key 去查询数据所处于链表数组的位置，定位到具体的链表，然后对链表进行加锁，因此如果链表数组的长度为 n，则最多允许 n 个线程同时插入数据。接下来简单的看下
ConcurrentHashMap 的插入方法的具体实现。

## 源码
```java

public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {

    /**
    * 存储数据的链表数组，使用 volatile 关键字修饰，保证内存可见性
    */
    transient volatile Node<K,V>[] table;

    /**
    * 插入数据
    */
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /**
    * 插入数据的实现方法
    */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        // 简述校验
        if (key == null || value == null) throw new NullPointerException();
        // 获取 key 的 hash 值
        int hash = spread(key.hashCode());
        int binCount = 0;
        // 死循环，将 table 赋值给临时变量
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 如果 map 为空则初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 如果 key 在链表数组中的位置为空，则直接使用 CAS 的方式进行插入数据
            // 如果成功跳出循环，否则则代表产生并发，然后进入下一次循环
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果在扩容，则更新 tab
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                // 当前链表加锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        // hash 值大于 0，说明是链表
                        if (fh >= 0) {
                            // 记录链表长度
                            binCount = 1;
                            // 遍历链表
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 判断 key 是否存在
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 插入新值
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 如果是 红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            // 插入数据
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // 判断是否需要转换为红黑树
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // map 大小 + 1
        addCount(1L, binCount);
        return null;
    }

}

```