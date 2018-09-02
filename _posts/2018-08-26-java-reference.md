---
layout: post
title: "Java Reference & ReferenceQueue"
date: 2018-08-22
tags: [Java, JVM, GC]
comments: true
share: true
---

## Reference
在 JDK 1.2 之前，Java 中的引用的定义很传统：如果 reference 类型的数据中存储的数值代表的是另一块内存的起始地址，就称这块内存代表着一个引用。这种定义很纯粹，但是太多狭隘，一个对象在这种定义下只有被引用活着没有被引用两种状态。在 JDK 1.2 之后，Java 对引用的概念进行了扩充，将引用分为强引用（Strong Reference）、软引用（Soft Reference）、弱引用（Weak Reference）、虚引用（Phantom Reference）4种，这 4 种引用强度一次逐渐减弱。

- 强引用就是指在程序代码之中普遍存在的，类似 “Object obj = new Object()” 这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象。

- 软引用使用来描述一些还有用但是非必须的对象，对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收返回之中进行第二次回收。如果这次回收还没有足够的内存，才会跑出内存溢出。在 JDK 1.2 之后，提供了 SoftReference 类来实现软引用。

- 弱引用也是用来描述非必须对象的，但是它的强度比软引用更弱一些，被弱引用关联的喜爱那个只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在 JDK 1.2 之后，提供了 WeakReference 类来实现弱引用。

- 虚引用也被成为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被垃圾回收器回收的时取得一个系统通知，用于做后续相关清理工作。在 JDK 1.2 之后，提供了 PhantomReference 类来实现虚引用。

类图如下。

![java ref structure](/images/java_ref_structure.png)

## ReferenceQueue
通过 Reference 的构造函数我们可以看到在初始化的时候会传入一个 ReferenceQueue ，而 ReferenceQueue 的作用就像是在 Reference 引用的对象被 GC 掉后，会把 Reference 这个对象放入到 ReferenceQueue 中做后续处理工作。代码示例如下：

```java
public class ReferenceQueueDemo {

    public static void main(String[] args) {
        ReferenceQueue<Bean> q = new ReferenceQueue<>();
        ExtendedWeakReference ref = new ExtendedWeakReference(new Bean("ref"), q);

        System.gc();

        Reference<? extends Bean> r;

        while((r = q.poll()) != null) {
            ExtendedWeakReference t = (ExtendedWeakReference) r;
            System.out.println("回收了 bean 对象，name 为:" + t.name);
            t.clean();
        }
    }


    static class Bean {
        String name;

        public Bean (String name) {
            this.name = name;
        }
    }

    static class ExtendedWeakReference extends WeakReference<Bean> {

        String name;

        public ExtendedWeakReference(Bean referent, ReferenceQueue<? super Bean> q) {
            super(referent, q);
            this.name = referent.name;
        }

        public void clean() {
            this.name = null;
        }

    }
}
```