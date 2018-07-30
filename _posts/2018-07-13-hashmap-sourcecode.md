---
layout: post
title: "HashMap 源码解读(JDK 1.8)"
date: 2018-07-13
tags: [Java,集合]
comments: true
share: true
---


## 概述
HashMap 是我们常用的集合之一，其数据是键值对的格式存储的，底层使用的链表数组实现的，当某个链表的长度达到一定长度时，会优先对数组链表进行扩容，当链表数组达到一定的长度后，便会将链表转换为红黑树。我们常使用的方法有的 put、get、containsKey、containsValue 等，接下来我们会通过源码解析来了解他是如何实现的。


## 相关名词
### capacity
链表数组长度，默认值 16，且**值必须为 2 的幂次方**，可以在创建 HashMap 时进行设置，如果创建 HashMap 时设置的不是 2 的幂次方，便会将数组长度设置为大于该值的 2 的幂次方。

### loadFactor
负载系数，通过该值来计算出扩容阈值（threshold）。

### threshold
扩容阈值，当 map 的大小（键值对的映射数量）大于等于该值的时候，便会对链表数组进行扩容，具体计算公式为 capacity * loadFactor = threshold

## 源码解析
```java
public class HashMap<K,V> extends AbstractMap<K,V> 
                            implements Map<K,V>, Cloneable, Serializable {
    /**
    * 链表数组默认初始化长度
    */
    static final int DEFAULT_INITIAL_CAPACITY = 16;

    /**
    * 链表数组的最大长度，达到该长度后，链表数组将不再进行扩容
    */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
    * 负载系数默认值
    */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
    * 链表转换为红黑树的阈值，如果某个链表的长度超过该阈值，
    * 链表便会转换为红黑树
    */
    static final int TREEIFY_THRESHOLD = 8;

    /**
    * 红黑树转换为链表的阈值，如果某个红黑树的节点个数超过该值，
    * 便会将红黑树转换为链表，例如移除数据导致某个红黑树的节点
    * 小于该值，便会将其转换为链表
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
    * 如果 key 不为空， key 的 hascode 记为 a，a 无符号位移 16 位记为 b，
    * 然后返回 a 跟 b 位异运算的值。
    * Java 各种位运算解析参考 https://blog.zhangyong.io/java-bit-operations/
    */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    /**
    * 存放数据的链表数组，本篇文章中所说的数组基本都是链表数组
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
    * 扩容阈值，当 map 的大小超过该值，便会对数组进行扩容
    */
    int threshold;

    /**
    * 负载系数，链表数组长度 * 负载系数 = 扩容阈值
    */
    final float loadFactor;

    /**
    * 构造方法，指定链表数组长度和负载系数，构造方法中没有直接初始化链表数组，
    * 而是通过 capacity 计算出了扩容阈值，然后在第一次 put 数据的时候，进行
    * 初始化链表数组长度，而长度直接等于扩容阈值。该构造函数只做了两件事：
    * 1. 设置负载系数。
    * 2. 根据是初始化的长度设置扩容阈值，扩容阈值始终是 大于等于 capacity 的
    *    2 的幂次方
    * 
    */
    public HashMap(int initialCapacity, float loadFactor) {
        // 参数校验
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        // 设置负载系数
        this.loadFactor = loadFactor;
        // 计算扩容阈值
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
    * 构造方法，传入一个另一个 Map，将数据导入到新生成的 Map 中
    */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        // 调用 putall 的实现方法
        putMapEntries(m, false);
    }

    /**
    * putall 的实现方法
    */
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        // 参数 map 的长度赋值给临时变量 s
        int s = m.size();
        // 判断是否是一个空 map
        if (s > 0) {
            // 如果是一个空 map，则计算出这个新 map 的扩容阈值，然后在第一次 put 数据
            // 的时候初始化链表数组，长度便是这个扩容阈值，这里初始化 map 的逻辑是跟调
            // 用指定 capacity 的构造方法的逻辑是一致的。
            if (table == null) {
                // 计算出新 map 的扩容阈值（也就是链表数组的长度）
                float ft = ((float)s / loadFactor) + 1.0F;
                // 限定最大不能超过长度阈值
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                // 如果新的扩容阈值大于当前扩容阈值，才会将当前扩容阈值设置为新的扩容阈值，
                // 防止链表数组长度缩短
                if (t > threshold)
                    // 得出正确的扩容阈值（链表数组长度），必须为 2 的幂次方
                    threshold = tableSizeFor(t);
            } else if (s > threshold) {
                // 如果不为空，且参数 map 的大小超过了扩容阈值，那么也进行扩容。
                resize();
            }
            // 循环调用 putVal 方法插入数据
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
            // 校验是是否是链表的头部，如果是进行查询的 key，则直接返回，否则进行链表遍历
            if (first.hash == hash &&
                ((k = first.key) == key || (key != null && key.equals(k)))) {
                return first;
            }
            // 赋值链表的下一个节点给临时变量，并校验链表的下一个不为空，
            // 如果为空则表示不存在该 key 不存在，直接返回 null
            if ((e = first.next) != null) {
                // 如果是红黑树，则遍历红黑树
                if (first instanceof TreeNode) {
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                }
                // 循环遍历整个链表
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
        // 根据 key 的 hash 值和数组长度计算数据所处数组中的位置，如果该下标下的链表为空，
        // 直接赋值
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
                // 如果是红黑树，则调用红黑树的节点的赋值方法进行赋值，如果 key 已存在，
                // 返回已存在的树节点 
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            } else {
                // 否则遍历链表
                for (int binCount = 0; ; ++binCount) {
                    // 首先赋值临时变量 e 为链表节点的下一个元素，然后判断如果遍历到链表底部
                    // （链表的下一个元素为空），则进行赋值，然后跳出循环
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 如果链表长度到达 7 个，将链表转换为红黑树，
                        if (binCount >= TREEIFY_THRESHOLD - 1) {
                            treeifyBin(tab, hash);
                        }
                        break;
                    }
                    // 如果 key 已经存在，将已存在的 key 对应 的节点赋值给临时变量 e，
                    // 然后跳出循环
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
                // HashMap 的该方法为空方法，没有实现，LinkedHashMap 中进行了实现
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
        // HashMap 的该方法为空方法，没有实现，LinkedHashMap 中进行了实现
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
        // 将扩容阈值赋值给临时变量
        int oldThr = threshold;
        // 声明新的长度跟扩容阈值对应的临时变量
        int newCap, newThr = 0;
        // 判断 map 中是否已经存在数据，如果存在数据，则数组长度大于0
        if (oldCap > 0) {
            // 如果长度达到设定的最大长度阈值阈值，则将扩容阈值设置为 Integer 的最大值。
            // 代表将不再会对数组进行扩容，返回原数组。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 设置新的长度，然后判断新的长度如果小于最大容量阈值并且当前长度小于默认长度，
            // 便设置新的扩容阈值，其新值都是旧值的两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY) {
                newThr = oldThr << 1;
            }
        }
        // 如果长度为0且扩容阈值大于0，直接将容量设置为扩容阈值
        else if (oldThr > 0)
            newCap = oldThr;
        // 如果都为0，则设置为默认值
        else { 
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 如果扩容阈值还没有进行初始化，初始化扩容阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            // 判断如果新的数组长度大于扩容上限阈值，则直接设置为 Integer 最大值，
            // 代表不再进行扩容，否则设置为已经计算出来的扩容阈值
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
        // 如果老的链表数组数据不为空，则进行迁数据移
        if (oldTab != null) {
            // 循环遍历就的链表数组
            for (int j = 0; j < oldCap; ++j) {
                // 临时变量
                Node<K,V> e;
                // 赋值给临时变量并判断该数组下标下的链表不为空
                if ((e = oldTab[j]) != null) {
                    // 删除该位置对于链表的引用，使其可以进行垃圾回收？
                    // 自己的理解，不确定是否正确。。。。。
                    oldTab[j] = null;
                    // 如果该位置下的链表长度唯一，则直接重新结算改节点在新数组中的
                    // 位置，并添加到新的数组中。Q:直接设置到该位置的头部节点？
                    // 如何确定该位置没有数据呢？
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 如果是红黑树，则将该节点下的子节点分散到新的数组中去
                        // 还没看方法实现，猜的~
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else {
                        // 链表优化重hash的代码块
                        // 声明临时变量
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            // 节点的下一个变量赋值给临时变量
                            next = e.next;
                            // 原索引
                            if ((e.hash & oldCap) == 0) {
                                // 第一次进来，设置头节点，否则设置尾节点
                                if (loTail == null) {
                                    loHead = e;
                                } else {
                                    loTail.next = e;
                                }
                                // 无论如何，都更新尾节点
                                loTail = e;
                            } else {
                                // 原索引+oldCap
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        
                        // 存放在原索引位置
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 存放在原索引+oldCap的位置里
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

    /**
     * 将链表数组转换为红黑树
     * @param tab 链表数组
     * @param hash 插入 key 的 hash 值
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        // 声明临时变量
        int n, index; Node<K,V> e;
        // 如果链表数组为空或者链表数组的长度小于阈值时，优先进行扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) {
            resize();
        } 
        // 计算 key 在链表数组中的具体下标，并校验该位置下的链表不为空
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            // 声明临时变量
            TreeNode<K,V> hd = null, tl = null;
            // 遍历整个链表，并设置红黑树的 pre next 相关节点
            do {
                // 将链表节点替换为红黑树节点
                TreeNode<K,V> p = replacementTreeNode(e, null);
                // 设置 pre next 相关节点
                if (tl == null) {
                    hd = p;
                } else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            // 把链表所处在数组的位置重新复制给新的红黑树 root 节点，
            // 并判断 root 节点不为空，然后将链表转换为红黑树。
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }

    /**
     * 将另一个 map 中的数据插入到当前 map 中
     *
     * @param m 另一个存储数据的 map
     * @throws NullPointerException 如果参数 map 为空
     */
    public void putAll(Map<? extends K, ? extends V> m) {
        // 调用实现方法
        putMapEntries(m, true);
    }

    /**
     * 移除指定 key 的键值对映射
     *
     * @param  key 即将被移除的 key
     * @return 如果 key 存在则返回 key 对应的 value，否则返回 null
     */
    public V remove(Object key) {
        // 被移除的节点
        Node<K,V> e;
        // 移除节点并返回节点的 value
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

    /**
     * Map.remove 的实现方法
     *
     * @param hash key 的 hash 值
     * @param key 即将被移除的 key
     * @param value 如果 matchValue 为 true，则当 key 与 value 同时一致时才移除
     * @param matchValue 如果为 true 则校验 key 对应的 value 与传入的是否一致
     * @param movable 如果为 false 则不一定其他节点
     * @return 如果 key 存在则返回被移除的节点，否则返回 null
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        //声明临时变量
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        // 如果当前链表数组不为空且 key 所在的数组下标对应的链表不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            // 校验链表头结点是否是要删除的节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            // 判断头结点的下一个节点不为空
            else if ((e = p.next) != null) {
                // 如果节点是红黑树节点，调用红黑树的查询方法
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    // 遍历链表，找到 key 相同的节点并跳出循环
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            // 如果被移除的节点不为并且在匹配 value 的情况下，value 相等，则删除节点
            // （这个判断写的属实绕！！！）
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                // 如果节点是红黑树节点，则调用红黑树的移出方法
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                // 如果移出的是链表的头结点，则直接指向头结点的下一个节点
                else if (node == p)
                    tab[index] = node.next;
                // 否则将 node 的上一个节点的下一个节点指向 node 的下一个节点
                else
                    p.next = node.next;
                // 修改次数加一
                ++modCount;
                // map 的键值对数量减一
                --size;
                // 移出数据后的回调，HashMap 是个空方法，LinkedHashMap 中有实现
                afterNodeRemoval(node);
                // 返回被移除的节点
                return node;
            }
        }
        return null;
    }


    /**
     * 红黑树，具有五个基本性质
     * 1. 每个节点只有一种颜色，要么红色，要么黑色
     * 2. 根节点是黑色
     * 3. 叶节点（空节点）是黑色
     * 4. 如果节点是红色，那么他的两个子节点都是黑色
     * 5. 任意的节点，从该节点到其所有后代叶节点的简单路径，均包含相同数目的黑色节点，
     * 每次新增或删除一个节点后，都有可能破坏红红黑树的这几个基本性质，因此每次操作后
     * 都需要对其进行自底向上的修复操作，修复操作分为两类：
     * 1. 重新着色
     * 2. 旋转操作
     * 该类还继承了 Entry 类，说明该类还具有链表相关的特性。
     */
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        /*
        * 父节点
        */ 
        TreeNode<K,V> parent;
        /*
        * 左子节点
        */ 
        TreeNode<K,V> left;
        /*
        * 右子节点
        */ 
        TreeNode<K,V> right;
        /*
        * 下一个节点（链表特点）
        */ 
        TreeNode<K,V> prev;
        /*
        * 节点颜色，true 红色，false 黑色
        */ 
        boolean red;

        /**
        * 构造函数，初始化一个树节点
        */
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }

        /**
         * 获取红黑树的根节点
         */
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }

        /**
         * 将红黑树的根节点与红黑树所处的链表数组的下标下存储的节点对应，并设置红黑树的根节点为
         * 链表的头结点
         */
        static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
            // 链表数组长度
            int n;
            // 参数校验，root 跟 链表数组均不为空
            if (root != null && tab != null && (n = tab.length) > 0) {
                // 重新计算红黑树 root 节点所处链表数组的下标位置
                int index = (n - 1) & root.hash;
                // 下标对应的链表头节点
                TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
                // 如果红黑树的根节点和下标下的链表头结点不一致
                if (root != first) {
                    // 红黑树根节点的下一个节点
                    Node<K,V> rn;
                    // 将下标对应的节点更改为红黑树的根节点
                    tab[index] = root;
                    // 红黑树根节点的上一个节点
                    TreeNode<K,V> rp = root.prev;
                    // 如果根节点的上一个节点不为空
                    if ((rn = root.next) != null)
                        // 根节点的下一个节点的上一个节点指向根节点的上一个节点，
                        // 相当于把根节点从链表中删除
                        ((TreeNode<K,V>)rn).prev = rp;
                    // 如果根节点的上一个节点上一个节点不为空
                    if (rp != null)
                        // 根节点的上一个节点的下一个节点指向根节点的下一个节点
                        rp.next = rn;
                    // 链表的头结点不为空
                    if (first != null)
                        // 链表的头结点指向红黑树的根节点
                        first.prev = root;
                    // 根节点的下个节点指向原链表的头结点
                    root.next = first;
                    // 根节点的上个节点设置为空
                    root.prev = null;
                }
                // 校验数据正常
                assert checkInvariants(root);
            }
        }

        /**
         * 红黑树查询实现方法方法
         */
        final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
            // 复制当前对象到临时变量
            TreeNode<K,V> p = this;
            do {
                // 初始化临时变量并赋值
                int ph, dir; K pk;
                TreeNode<K,V> pl = p.left, pr = p.right, q;
                // 如果查询的 hash 小于当前节点的 hash 值
                // 则将左子树赋值给临时变量，继续循环查找
                if ((ph = p.hash) > h)
                    p = pl;
                // 如果查询的 hash 大于当前节点的 hash 值
                // 则将右子树赋值给临时变量，继续循环查找
                else if (ph < h)
                    p = pr;
                // 如果 key 相等，如果值相等， 则返回
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                // 如果左子树为空，则将右子树赋值给临时变量
                else if (pl == null)
                    p = pr;
                // 如果右子树为空，则将左子树赋值给临时变量
                else if (pr == null)
                    p = pl;
                // 判断 k 是否是 Comaprable 类型，如果是继续判断当前节点的
                // key 跟 k 是否是同一类型，如果是则调用 compareTo 方法
                // 比较两值的大小
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                // 以上情况都不满足的情况下则去右子树节点下查找
                else if ((q = pr.find(h, k, kc)) != null)
                    return q;
                // 左子树节点下查找
                else
                    p = pl;
            } while (p != null);
            return null;
        }

        /**
         * 通过 key 和 key 的 hash 值查询红黑树节点
         */
        final TreeNode<K,V> getTreeNode(int h, Object k) {
            return ((parent != null) ? root() : this).find(h, k, null);
        }

        /**
         * 在 hashCode 相等且 key 均没有实现 Comparable 进行比较的情况下，
         * 调用该方法比较两个 key 的大小
         */
        static int tieBreakOrder(Object a, Object b) {
            int d;
            if (a == null || b == null ||
                (d = a.getClass().getName().
                 compareTo(b.getClass().getName())) == 0)
                d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                     -1 : 1);
            return d;
        }

        /**
         * 将链表转换为红黑树
         */
        final void treeify(Node<K,V>[] tab) {
            // 初始化 root 节点的临时变量
            TreeNode<K,V> root = null;
            // 循环当前红黑树，设置红黑树的相关属性
            for (TreeNode<K,V> x = this, next; x != null; x = next) {
                // 赋值下一个节点给 next 变量
                next = (TreeNode<K,V>)x.next;
                // 初始化当前节点的左右子节点
                x.left = x.right = null;
                // 如果 root 为空（第一次遍历），将当前对象设置为红黑树的根节点
                if (root == null) {
                    // 根节点的父节点为空
                    x.parent = null;
                    // 根节点为黑色
                    x.red = false;
                    // x 赋值给 root 节点变量
                    root = x;
                }
                else {
                    // 将当前节点的 key 和 hash 赋值给临时变量
                    K k = x.key;
                    int h = x.hash;
                    // key 的 Class 对象
                    Class<?> kc = null;
                    // 将当前节点插入到红黑树中，对 putTreeVal 方法进行了简化，
                    // 因为是第一次插入，不会再去查询 key 是否已经在红黑树中存在了，
                    // 进而提升一定的效率
                    for (TreeNode<K,V> p = root;;) {
                        int dir, ph;
                        K pk = p.key;
                        if ((ph = p.hash) > h)
                            dir = -1;
                        else if (ph < h)
                            dir = 1;
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            dir = tieBreakOrder(k, pk);

                        TreeNode<K,V> xp = p;
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {
                            x.parent = xp;
                            if (dir <= 0)
                                xp.left = x;
                            else
                                xp.right = x;
                            root = balanceInsertion(root, x);
                            break;
                        }
                    }
                }
            }
            // 确保红黑树根节点与红黑树所处的链表数组的下标下存储的节点对应
            // （保证每次搜索时是从红黑树的根节点去搜索啊！！！）
            moveRootToFront(tab, root);
        }

        /**
         * 将红黑树转换为链表
         */
        final Node<K,V> untreeify(HashMap<K,V> map) {
            Node<K,V> hd = null, tl = null;
            for (Node<K,V> q = this; q != null; q = q.next) {
                Node<K,V> p = map.replacementNode(q, null);
                if (tl == null)
                    hd = p;
                else
                    tl.next = p;
                tl = p;
            }
            return hd;
        }

        /**
         * 红黑树版本的插入数据实现方法
         */
        final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
            // 参数 k 的 Class 类引用
            Class<?> kc = null;
            // 当前节点以下所有节点是否都对参数 k 进行了查询
            boolean searched = false;
            // 将 root 节点赋值给临时变量
            TreeNode<K,V> root = (parent != null) ? root() : this;
            // 将 root 节点作为查询的初始化节点
            for (TreeNode<K,V> p = root;;) {
                // 声明临时变量
                int dir, ph; K pk;
                // 赋值当前节点的 hash 值给临时变量，并与要插入的
                // k 的 hash 值进行比较，并将比较结果赋值给临时变量
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                // 此时 hash 值应该是相等的，再去比较如果当前节点的 key 与插入的 key 相等，
                // 则返回当前节点，因为有可能会出现 hash 值相等但 key 不相同的情况，
                // 因此没有单纯的直接使用 hash 去比较是否相等，而是直接因为这样可能会出现 bug
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                // 在通过 hash 值跟 k 比较不出的情况下，则判断插入的 key 是否
                // 是 Comparable 类型，以及两个类是否属于同一类型，如果是则
                // 调用 compareTo 比较两个 key 的大小，如果无法比较则从当前
                // 节点开始，查询所有子节点是否存在 key 相等的情况，如果有就
                // 直接返回。
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    // 如果以前没有进行过搜索，则查询当前节点下所有子节点的 key 
                    // 跟插入的 key 是否一致，如果一致则直接返回原节点
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        // 无论是否查到，都赋值给 searched 标识为已经查询过
                        searched = true;
                        // 从当前节点的左右子节点开始查找是否有子节点跟
                        // key 相等，如果有则直接返回
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    // 在 key 跟当前节点的 key 都不为空的情况下，通过类名去
                    // 比较大小，否则通过调用 System.identityHashCode 生
                    // 成的 hash 值去比较。Q:System.identityHashCode 的
                    // 具体作用？
                    dir = tieBreakOrder(k, pk);
                }

                // 将当前节点赋值给临时变量
                TreeNode<K,V> xp = p;
                // 根据比较结果来判断应该是插入到左右子节点中的哪一个，并判断
                // 其是否为空，如果为空则插入一个新的节点，否则更新循环条件，继续比较
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    // 将当前节点的下一个节点赋值给临时变量
                    Node<K,V> xpn = xp.next;
                    // 构建一个新节点
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                    // 如果 key 小于当前节点的 key 则赋值给左子树，赋值赋值
                    // 给右子树
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 将当前节点的下一节点更新为新插入的节点
                    xp.next = x;
                    // 将新插入的节点的父节点、上一个节点赋值为当前节点
                    x.parent = x.prev = xp;
                    // 更新 xpn 的上一个节点（真tm绕）
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    
                    moveRootToFront(tab, balanceInsertion(root, x));
                    return null;
                }
            }
        }

        /**
         * 删除给定的节点
         */
        final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable) {
            int n;
            // 参数校验
            if (tab == null || (n = tab.length) == 0)
                return;
            // 当前节点在数组中的下标
            int index = (n - 1) & hash;
            // 声明链表的头结点跟红黑树的头节点，红黑树的头结点的左子树
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
            // 当前节点的下一个节点和上一个节点
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
            // 如果当前节点头结点，直接将数组下标对应的节点改成当前节点的下一个节点
            if (pred == null)
                tab[index] = first = succ;
            // 否则上一个节点的下一个节点改成当前节点的下一个节点
            else
                pred.next = succ;
            // 如果下一个节点不为空，将下一个节点的上一个节点改成当前节点的上一个节点
            if (succ != null)
                succ.prev = pred;
            // 如果链表的头结点为空，则直接返回
            if (first == null)
                return;
            // 如果 root 节点不是红黑树的根节点，则赋值为红黑树的根节点
            if (root.parent != null)
                root = root.root();
            // 如果根节点为空或者根节点的左右子节点任意一个为空或者左子节点的
            // 左子节点为空，则代表红黑树数据量太小，转换为链表
            if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                tab[index] = first.untreeify(map);
                return;
            }
            // p 被删除的节点，pl p的左子节点，pr p的右子节点，
            // replacement 在被删除节点的做右子节点都不为空的情况下，是被删除节点的后继节点的后继节点
            // 否则便是被删除节点的后继节点
            TreeNode<K,V> p = this, pl = left, pr = right, replacement;
            // 如果左右子节点都不为空，寻找后继节点（右子树的最小值或者左子树的最大值为后继节点）和后继节点的
            // 后继节点，并将后继节点与被删除节点交换位置
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                // 循环遍历右子树，获得最小值为后继节点
                while ((sl = s.left) != null)
                    s = sl;
                // 交换颜色
                boolean c = s.red; s.red = p.red; p.red = c;
                // 后继节点的右子节点
                TreeNode<K,V> sr = s.right;
                // 被删除节点的父节点
                TreeNode<K,V> pp = p.parent;
                // 如果被删除的节点是后继节点的父节点
                if (s == pr) {
                    // 将被删除节点的父节点设置为后继节点
                    p.parent = s;
                    // 后继节点的右子节点设置为被删除节点
                    s.right = p;
                }
                else {
                    // 后继节点的父节点
                    TreeNode<K,V> sp = s.parent;
                    // 被删除节点的父节点设置为后继节点的父节点，并判断不为空
                    if ((p.parent = sp) != null) {
                        // 如果后继节点是父节点的左子节点，则替换为被删除的节点
                        // 否则将右子节点替换为被删除的节点
                        if (s == sp.left)
                            sp.left = p;
                        else
                            sp.right = p;
                    }
                    // 后继节点的右子节点设置为被删除节点的右子节点
                    if ((s.right = pr) != null)
                        // 并将右子节点的父子节点设置为后继节点
                        pr.parent = s;
                }
                // 因为后继节点是被删除节点的右子树的最小值，所以后继节点肯定没有左子节点，
                // 因此直接将被删除节点的左子节点设置为空
                p.left = null;
                // 将被删除节点的右子节点替换为后继节点的右子节点
                if ((p.right = sr) != null)
                    sr.parent = p;
                // 将后继节点的左子节点设置为被删除节点的左子节点
                if ((s.left = pl) != null)
                    pl.parent = s;
                // 将后继节点的父节点设置为被删除节点的父节点，如果父节点为空
                // 则将后继节点设置为红黑树的根节点
                if ((s.parent = pp) == null)
                    root = s;
                // 如果被删除节点是原父节点的左子节点，则将后继节点设置为父节点的左子节点
                // 否则设置为右子节点
                else if (p == pp.left)
                    pp.left = s;
                else
                    pp.right = s;
                // 因为后继节点不存在左子节点，所以直接判断右子节点是否为空，如果右子节点不为空
                // 右子节点来替换后继节点，否则被删除节点替换后继节点
                if (sr != null)
                    replacement = sr;
                else
                    replacement = p;
            }
            // 如果只有左子节点不为空，则将后继节点设置为左子节点
            else if (pl != null)
                replacement = pl;
            // 如果只有右子节点不为空，则将后继节点设置为右子节点
            else if (pr != null)
                replacement = pr;
            // 否则后继节点就是被删除节点本身
            else
                replacement = p;
            // 如果后继节点不是被删除节点本身
            if (replacement != p) {
                // 后继节点的父节点设置为被删除节点的父节点
                TreeNode<K,V> pp = replacement.parent = p.parent;
                // 如果父节点为空，则将根节点设置为后继节点
                if (pp == null)
                    root = replacement;
                // 将父节点与后继节点关联（此时已经将被删除的节点从书中删除）
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement; 
                // 删除被删除节点对父节点、左右子节点的引用
                p.left = p.right = p.parent = null;
            }
            
            // 根据被删除节点的颜色，判断是否修复红黑树的性质，因为如果被删除的节点是黑色，
            // 则可能导致该路径比其他路径少一颗黑色节点从而破坏红黑树性质，因此需要修复红黑树
            TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

            // 如果后继节点为被删除结点本身，删除与父节点的相互引用
            if (replacement == p) {
                TreeNode<K,V> pp = p.parent;
                // 删除对父节点的引用
                p.parent = null;
                // 父节点删除对被删除节点的引用
                if (pp != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                }
            }
            // 如果为 true， 确保红黑树根节点与红黑树所处的链表数组的下标下存储的节点对应
            if (movable)
                moveRootToFront(tab, r);
        }

        /**
         * Splits nodes in a tree bin into lower and upper tree bins,
         * or untreeifies if now too small. Called only from resize;
         * see above discussion about split bits and indices.
         *
         * @param map the map
         * @param tab the table for recording bin heads
         * @param index the index of the table being split
         * @param bit the bit of hash to split on
         */
        final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }

        /**
        * 左旋操作
        */
        static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            if (p != null && (r = p.right) != null) {
                if ((rl = p.right = r.left) != null)
                    rl.parent = p;
                if ((pp = r.parent = p.parent) == null)
                    (root = r).red = false;
                else if (pp.left == p)
                    pp.left = r;
                else
                    pp.right = r;
                r.left = p;
                p.parent = r;
            }
            return root;
        }

        /**
        * 右旋操作
        */
        static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            if (p != null && (l = p.left) != null) {
                if ((lr = p.left = l.right) != null)
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false;
                else if (pp.right == p)
                    pp.right = l;
                else
                    pp.left = l;
                l.right = p;
                p.parent = l;
            }
            return root;
        }

        /**
        * 插入后修复红黑树性质
        */
        static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
            // 新插入的节点为空色
            x.red = true;
            // 初始化相关变量
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                // xp 赋值为 x 节点的父节点，并且判断如果 x 的父节点为空
                // 则表示 x 为红黑树的根节点，将 x 节点设置为黑色，并返回
                if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                // 将 xp 的父节点赋值为 xpp（xpp 为插入节点即 x 节点的爷爷节点），
                // 如果 xp 为黑色或者 xpp 节点为空，则代表 xp 为根节点，
                // 因此不需要进行修复，直接返回
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;
                // 如果父节点为爷爷节点的左子树
                if (xp == (xppl = xpp.left)) {
                    // 如果爷爷节点的右子树不为空且也为红色，此时 x 的节点为
                    // 红色且父节点跟叔叔节点也为红色，不符合红黑树性质，进行
                    // 重新着色修复
                    if ((xppr = xpp.right) != null && xppr.red) {
                        // x 的叔叔节点跟父节点设置为黑色
                        xppr.red = false;
                        xp.red = false;
                        // x 的爷爷节点设置为红色
                        xpp.red = true;
                        // 将 x 更新为爷爷节点
                        x = xpp;
                    }
                    else {
                        // 如果叔叔节点为空或者为黑色，且 x 节点为父节点的右子节点
                        // 则先进行左旋操作
                        if (x == xp.right) {
                            // 左旋操作
                            root = rotateLeft(root, x = xp);
                            // 更新爷爷节点
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        // 如果父节点不为空
                        if (xp != null) {
                            // 父节点设置为黑色
                            xp.red = false;
                            // 如果爷爷节点不为空
                            if (xpp != null) {
                                // 爷爷节点设置为红色
                                xpp.red = true;
                                // 进行右旋操作
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                // 如果父节点为爷爷的节点的右子节点
                else {
                    // 如果叔叔节点不为空且叔叔节点也为红色，
                    // 直接进行重新着色来修复
                    if (xppl != null && xppl.red) {
                        // 叔叔节点跟父节点设置为黑色
                        xppl.red = false;
                        xp.red = false;
                        // 爷爷节点设置为红色
                        xpp.red = true;
                        // x 节点更新为爷爷节点
                        x = xpp;
                    }
                    // 如果叔叔节点为空或者为黑色
                    else {
                        // 如果当前节点为父节点的左子节点
                        // 先进行右旋操作
                        if (x == xp.left) {
                            // 左旋操作
                            root = rotateRight(root, x = xp);
                            // 更新爷爷节点
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        // 如果父节点不为空
                        if (xp != null) {
                            // 父节点设置为黑色
                            xp.red = false;
                            // 爷爷节点不为空，则进行左旋操作
                            if (xpp != null) {
                                // 爷爷节点设置为红色
                                xpp.red = true;
                                // 左旋操作
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }

        /**
        * 删除节点后修复红黑树的性质，参数 x 黑色
        */
        static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                                   TreeNode<K,V> x) {
            // 自底向上修复红黑树
            for (TreeNode<K,V> xp, xpl, xpr;;)  {
                // 如果 x 节点为空或者为根节点，则停止修复
                if (x == null || x == root)
                    return root;
                // 如果 x 节点为根节点，直接将其设置为黑色
                else if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                // 如果为红色，设置为黑色
                else if (x.red) {
                    x.red = false;
                    return root;
                }
                // 如果 x 节点是父节点的左子节点
                else if ((xpl = xp.left) == x) {
                    // //有红色的兄弟节点xpr，则父亲节点xp必为黑色
                    if ((xpr = xp.right) != null && xpr.red) {
                        // 兄弟节点设置为黑色
                        xpr.red = false;
                        // 父节点设置为红色
                        xp.red = true;
                        // 父节点左旋
                        root = rotateLeft(root, xp);
                        // 更新 xp 为 x 的父节点，xpr 为 xp 的右子节点
                        xpr = (xp = x.parent) == null ? null : xp.right;
                    }
                    // 如果兄弟节点为空
                    if (xpr == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                        // 如果原叔叔树节点的左右子树都不为空切且为黑色，则将原叔叔节点
                        // 设置为红色，更新 x 节点为原父节点
                        if ((sr == null || !sr.red) &&
                            (sl == null || !sl.red)) {
                            xpr.red = true;
                            x = xp;
                        }
                        else {
                            // 如果原叔叔节点的右子树为空，或者为黑色
                            if (sr == null || !sr.red) {
                                // 如果左子节点不为空，左子节点也设置为黑色
                                if (sl != null)
                                    sl.red = false;
                                // 原叔叔节点设置为红色
                                xpr.red = true;
                                // 原叔叔节点右旋
                                root = rotateRight(root, xpr);
                                xpr = (xp = x.parent) == null ?
                                    null : xp.right;
                            }
                            if (xpr != null) {
                                xpr.red = (xp == null) ? false : xp.red;
                                if ((sr = xpr.right) != null)
                                    sr.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateLeft(root, xp);
                            }
                            x = root;
                        }
                    }
                }
                // 如果 x 节点是父节点的右子节点，对称操作
                else {
                    if (xpl != null && xpl.red) {
                        xpl.red = false;
                        xp.red = true;
                        root = rotateRight(root, xp);
                        xpl = (xp = x.parent) == null ? null : xp.left;
                    }
                    if (xpl == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                        if ((sl == null || !sl.red) &&
                            (sr == null || !sr.red)) {
                            xpl.red = true;
                            x = xp;
                        }
                        else {
                            if (sl == null || !sl.red) {
                                if (sr != null)
                                    sr.red = false;
                                xpl.red = true;
                                root = rotateLeft(root, xpl);
                                xpl = (xp = x.parent) == null ?
                                    null : xp.left;
                            }
                            if (xpl != null) {
                                xpl.red = (xp == null) ? false : xp.red;
                                if ((sl = xpl.left) != null)
                                    sl.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateRight(root, xp);
                            }
                            x = root;
                        }
                    }
                }
            }
        }

        /**
         * 校验节点既符合链表特点也符合红黑树特点
         */
        static <K,V> boolean checkInvariants(TreeNode<K,V> t) {
            TreeNode<K,V> tp = t.parent, tl = t.left, tr = t.right,
                tb = t.prev, tn = (TreeNode<K,V>)t.next;
            if (tb != null && tb.next != t)
                return false;
            if (tn != null && tn.prev != t)
                return false;
            if (tp != null && t != tp.left && t != tp.right)
                return false;
            if (tl != null && (tl.parent != t || tl.hash > t.hash))
                return false;
            if (tr != null && (tr.parent != t || tr.hash < t.hash))
                return false;
            if (t.red && tl != null && tl.red && tr != null && tr.red)
                return false;
            if (tl != null && !checkInvariants(tl))
                return false;
            if (tr != null && !checkInvariants(tr))
                return false;
            return true;
        }
    }

}
```