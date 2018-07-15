---
layout: post
title: "Java 集合体系学习"
description: "List、Set、Queue、Map"
date: 2018-07-10
tags: [Java,集合]
comments: true
share: true
---

## 概述
Java 最常见的四种集合为
- List
- Set
- Queue
- Map

其中，List、Set、Queue 均实现了 Collection 接口，而 Collection 又继承自 Iterable 接口，也就是说，List、Set、Queue 均是可以进行遍历的。而 Map 则是**键值对**形式的数据结构。

## List
### 特点
- 有序
- 可重复
- 提供了按索引查询的方法

### 相关子类及比较
#### ArrayList
- 继承了 AbstractList 类并实现了 List、RandomAccess、Clonable、java.io.Serializable 接口
- 数组实现
- 查询快，增、删、改慢
- 线程不安全

#### Vector
- 继承了 AbstractList 类并实现了 List、RandomAccess、Cloneable、java.io.Serializable 接口
- 数组实现
- 查询快，增、删、改慢
- 线程安全，每个方法都添加了 synchronized 关键字保证线程安全

#### LinkedList
- 继承了 AbstractSequentialList 类并实现了 List、Deque、Cloneable、java.io.Serializable 接口，而 AbstractSequentialList 类则继承自 AbstractList 类
- 双向链表实现
- 查询慢、增、删、改快
- 线程不安全

## Set
- 无序
- 不可重复

### 相关子类及比较
#### HashSet
- extends AbstractSet implements Set
- HashMap 实现
- 无序
- 线程不安全

#### LinkedHashSet
- extends HashSet implements Set 
- LinkedHashMap 实现
- 无序
- 线程不安全

#### TreeSet
- extends AbstractSet implements NavigableSet
- TreeMap 实现
- 通过 Comparator 实现排序
- 线程不安全

### Queue
### 特点
- 有序，先进先出
- 从查询与写入的角度来看，分为阻塞和非阻塞队列
- 从操作方式来看，分为单向操作跟双向操作（Queue&Deque）
- 从优先级的角度来看，还分为优先级与非优先级队列

### 相关子类及比较
#### ConcurrentLinkedQueue
- 非阻塞队列
- 线程安全
- 链表实现
- 先进先出，无优先级
- 单向操作

#### PriorityQueue
- 非阻塞队列
- 线程不安全
- 数组实现
- 通过 Comparator 实现排序优先级
- 单向操作

#### LinkedBlockingQueue
- 阻塞队列
- 线程安全
- 链表实现
- 先进先出，无优先级
- 单向操作

#### ArrayBlockingQueue
- 阻塞队列
- 线程安全
- 数组实现
- 先进先出，无优先级
- 单向操作

#### PriorityBlockingQueue
- 阻塞队列
- 线程安全
- 数组实现
- 通过 Comparator 实现排序优先级
- 单向操作

#### DelayQueue
- 阻塞队列
- 线程安全
- 封装 PriorityQueue 实现
- 先进先出，无优先级
- 单向操作

#### LinkedTransferQueue
//TODO

#### ArrayDeque
- 非阻塞队列
- 线程不安全
- 数组实现
- 双向操作，从头部或尾部开始操作

#### ConcurrentLinkedDeque
- 非阻塞队列
- 线程安全
- 双向链表实现
- 双向操作，从头部或尾部开始操作

#### LinkedBlockingDeque
- 阻塞队列
- 线程安全
- 双向链表实现
- 双向操作，从头部或尾部开始操作

## Map
### 特点
- 键值对数据结构

### 相关子类及比较
#### [HashMap](/hashmap-sourcecode)
- 数组链表实现
- 线程不安全
- key 允许为空

#### LinkedHashMap

#### WeakHashMap

#### Hashtable

#### TreeMap

<!-- ## QA
1. List 为例，因为 AbstractList 已经实现了 List 接口，而 ArrayList 已经继承了 AbstractList 抽象类，为什么 ArrayList 还要声明实现 List 接口？

    答：见 [ArrayList 为什么要重复声明实现 List 接口]()

2. 啦啦啦 -->