---
layout: post
title: "HashMap 源码解读"
description: "HashMap 源码解读"
date: 2018-07-13
tags: [Java]
comments: true
share: true
---

HashMap 是我们常用的集合之一，其数据是键值对的格式存储的，我们常使用的方法有的 put、get、containsKey、containsValue 等，接下来我们会通过源码解析来了解他是如何实现的。以下代码会对源码进行一定的拆分书写


```java
public class HashMap<K,V> extends AbstractMap<K,V> 
                            implements Map<K,V>, Cloneable, Serializable {
    /**
    * 数组默认初始化长度
    */
    static final int DEFAULT_INITIAL_CAPACITY = 16;

    /**
    * 数组最大长度？？？
    */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
    * 默认扩容阈值倍数，当数组的被使用长度超过数组的总长度的 0.75 倍时，
    * 便会进行扩容
    */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
    * 当链表数组某个下标下的链表长度超过定义的阈值时，会将改下标下的链表转换为
    * 红黑树
    */
    static final int TREEIFY_THRESHOLD = 8;

    /**
    * 如果红黑树的元素个数小于该阈值，会将红黑树转换为链表，重新扩容时会用到该阈值
    */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
    * HashMap 存储数据使用的链表类
    */
    static class Node<K,V> implements Map.Entry<K,V> {
        /**
        * key 的 hash 值
        */
        final int hash;
        /**
        * map 的 key 
        */
        final K key;
        /**
        * map 的 value
        */
        V value;
        /**
        * 链表的下一个节点
        */
        Node<K,V> next;

        /**
        * 构造方法
        */
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

    /**
    * 获取 key 的哈希值，具体逻辑如下
    * 如果 key 为空，返回 0 
    * 如果 key 不为空， key 的 hascode 记为 a，a 无符号位移 16 位记为 b，然后返回 a 跟 b 位异运算的值
    * Java 各种位运算解析参考 https://blog.zhangyong.io/java-bit-operations/
    */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    /**
    * 存放数据的链表数组
    */
    transient Node<K,V>[] table;

    /**
    * 键值对映射个数，map 容量
    */
    transient int size;
    
    /**
    * 内部结构被改变的次数，例如 修改了键值对的映射个数或者以其他方式修改了内部结构，
    * 例如重新hash，改字段主要用于在进行遍历时快速失败。（渣渣翻译，可否准确........）
    */
    transient int modCount;
    
    /**
    * 扩容阈值，当数组长度达到该变量地址的长度时，便会对数组进行扩容
    */
    int threshold;

    /**
    * 负载系数，链表数组长度 * 负载系数 = 扩容阈值
    */
    final float loadFactor;

    /**
    * 构造方法，指定链表数组长度和负载系数
    */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    /**
    * 构造方法，指定链表数组长度
    */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
    * 默认构造方法，所有成员变量均为默认值
    */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }

    /**
    * 构造方法，传入一个另一个 Map，将数据导入到该 Map 中
    */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        // 导入 参数map 中的数据到该 map 中
        putMapEntries(m, false);
    }

    /**
    * 将一个 map 中的数据导入到该 map 中
    */
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        // 参数 map 的长度赋值给临时变量 s
        int s = m.size();
        // 判断是否是一个空 map
        if (s > 0) {
            // 如果链表数组为空，初始化数组长度，提前设置数组长度，避免多次扩容而浪费性能
            if (table == null) {
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

    /**
    * 获取键值对映射个数
    */
    public int size() {
        return size;
    }

    /**
    * 判断 map 是否为空
    */
    public boolean isEmpty() {
        return size == 0;
    }

    /**
    * 通过 key 获取值
    */
    public V get(Object key) {
        Node<K,V> e;
        // 将 key 进行 hash，然后通过 key 的 hash 值跟 key 获取链表节点，
        // 然后返回链表节点中存储 value
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
    * 通过 key 的 hash 值跟 key 获取链表节点
    * @param hash key 的 hah 值
    * @param key key
    * @return key 对应的链表节点
    */
    final Node<K,V> getNode(int hash, Object key) {
        // 当前链表数组
        Node<K,V>[] tab; 
        // key 所在数组位置中对应的链表头部
        Node<K,V> first,
        // 循环遍历链表时用到的临时变量
        Node<K,v> e;
        // 当前链表数组的长度
        int n; 
        // 临时变量 key
        K k;
        // 将当前链表数组及其长度赋值给临时变量，并检验数组不为空，以及所在数组位置的链表不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 校验是是否是链表的头部，如果是 直接返回，否则进行链表遍历
            if (first.hash == hash &&
                ((k = first.key) == key || (key != null && key.equals(k)))) {
                return first;
            }
            // 校验链表的下一个不为空，如果为空则表示不存在该 key 不存在，直接返回 null
            if ((e = first.next) != null) {
                // 如果是红黑树，则遍历红黑树
                if (first instanceof TreeNode) {
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                }
                // 循环遍历链表，直到链表底部
                do {
                    // 如果 hash 相等且 key 相等，则返回该节点
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

    /**
    * 判断 key 是否存在
    */
    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }

    /**
     * 插入数据
     *
     * @param key 被插入数据的 key
     * @param value 被插入数据的值
     * @return 如果key存在返回key对应的旧值，否则返回 null
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * 插入数据实现
     *
     * @param hash key 的 hash 值
     * @param key 
     * @param value alue
     * @param onlyIfAbsent 如果为 true，不替换已存在的值
     * @param evict 如果为 false，代表 链表数组 处于创建模式
     * @return 如果 key 存在，返回旧值，否则返回空
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        // 当前的链表数组备份
        Node<K,V>[] tab; 
        // 临时变量，链表数组下标对应的链表的第一个元素
        Node<K,V> p; 
        // 当前链表数组的长度
        int n,
        // 数据在链表数组中所处的位置，即下标
        int i;
        // 如果是第一次执行，初始化链表数组的长度
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 计算数据所处数组中的位置，如果该下标下的链表为空，直接赋值
        if ((p = tab[i = (n - 1) & hash]) == null) 
            tab[i] = newNode(hash, key, value, null);
        else {
            // 如果 key 已存在对应的链表节点
            Node<K,V> e;
            K k;
            // 如果插入的数据的 key 的 hash 值与存放位置的链表的头元素的 hash 值相同，
            // 且 key 相等，则赋值给 变量 e
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k)))) {
                e = p;
            } else if (p instanceof TreeNode) {
                // 如果是红黑树，则调用红黑树的节点的赋值方法进行赋值，如果 key 已存在，返回已存在的树节点 
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            } else {
                // 否则遍历链表
                for (int binCount = 0; ; ++binCount) {
                    // 首先赋值临时变量 e 为链表节点的下一个元素，然后判断如果遍历到链表底部（链表的下一个元素为空），
                    // 则进行赋值，然后跳出循环
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 如果链表长度到达 7 个，将链表转换为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) {
                            treeifyBin(tab, hash);
                        }
                        break;
                    }
                    // 如果不是链表底部，判断如果 key 已经存在，将已存在的 key 对应的节点赋值给临时变量 e，然后跳出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))){
                        break;
                    }        
                    // 更新变量 p 为 p 的下个节点
                    p = e;
                }
            }
            // 如果临时 e 不为空，表示 key 已经存在
            if (e != null) {
                // 备份 key 对应的旧值
                V oldValue = e.value;
                // 如果 onlyIfAbsent 为 false 或者旧值为空，则替换为新值
                if (!onlyIfAbsent || oldValue == null) {
                    e.value = value;
                }
                // HashMap 的该方法为空方法，没有实现
                afterNodeAccess(e);
                // 返回旧值
                return oldValue;
            }
        }
        // 记录修改次数的变量加1
        ++modCount;
        // 容量加1，如果超过扩容阈值，进行扩容
        if (++size > threshold) {
            resize();
        }
        // HashMap 的该方法为空方法，没有实现
        afterNodeInsertion(evict);
        return null;
    }

    /**
    * 重新设置链表数组的长度
    *
    * @return 新长度的链表数组
    */
    final Node<K,V>[] resize() {
        // 将链表数组，数组长度，扩容阈值赋值给临时变量进行备份
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        // 声明新的长度跟扩容阈值对应的临时变量
        int newCap, newThr = 0;
        // 判断是否是第一次添加数据，第一次添加数据时，长度为 0
        if (oldCap > 0) {
            // 如果长度达到设定的最大长度阈值阈值，则将扩容阈值设置为 Integer 的最大值。
            //（代表将不再会对数组进行扩容），返回原数组。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 设置新的容量，然后判断新的长度如果小于最大容量阈值并且当前长度小于默认长度，
            // 则设置新的扩容阈值
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY) {
                newThr = oldThr << 1; // double threshold
            }
        }
        // 如果长度为0其扩容阈值大于0，直接将容量设置为扩容阈值
        else if (oldThr > 0)
            newCap = oldThr;
        // 如果都为0，则设置为默认值
        else { 
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 如果新的扩容阈值为0，则重新设置
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        // 赋值给成员变量扩容阈值
        threshold = newThr;
        // 创建一个新长度的数组
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        // 赋值给链表数组
        table = newTab;
        // 如果老的链表数组数据不为空，则进行迁移
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

}
```