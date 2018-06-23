---
layout: post
title: "this & ClassName.this"
date: 2017-06-14
tags: [Java]
comments: true
share: true
---

## 概述

* `this`  代表的是当前实例
* `ClassName.this` 代表的是也是 `ClassName` 这个类的当前实例。经常会出现在 jdk 源码中，当一个内部类需要调用外部类的实例时。

## 栗子

```java
public abstract class AbstractMap<K, V> implements Map<K, V> {

    public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            ks = new AbstractSet<K>() {
                public Iterator<K> iterator() {
                    return new Iterator<K>() {
                        private Iterator<Entry<K,V>> i = entrySet().iterator();

                        public boolean hasNext() {
                            return i.hasNext();
                        }

                        public K next() {
                            return i.next().getKey();
                        }

                        public void remove() {
                            i.remove();
                        }
                    };
                }

                public int size() {
                    return AbstractMap.this.size();
                }

                public boolean isEmpty() {
                    return AbstractMap.this.isEmpty();
                }

                public void clear() {
                    AbstractMap.this.clear();
                }

                public boolean contains(Object k) {
                    return AbstractMap.this.containsKey(k);
                }
            };
            keySet = ks;
        }
        return ks;
    }

}
```
