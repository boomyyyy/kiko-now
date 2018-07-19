---
layout: post
title: "HashMap 源码解读---基于 JDK 1.8 版本"
description: "HashMap 源码解读"
date: 2018-07-13
tags: [Java,集合]
comments: true
share: true
---


## 概述
HashMap 是我们常用的集合之一，其数据是键值对的格式存储的，我们常使用的方法有的 put、get、containsKey、containsValue 等，接下来我们会通过源码解析来了解他是如何实现的。以下代码会对源码进行一定的拆分书写


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
    * 存放数据的链表数组，本篇文章中所说的数组都是链表数组
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
     * 红黑树，继承了 LinkedHashMap 的 Entry 类，其实红黑树也具备了链表
     * 的特点
     */
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }

        /**
         * Returns root of tree containing this node.
         */
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }

        /**
         * Ensures that the given root is the first node of its bin.
         */
        static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
            int n;
            if (root != null && tab != null && (n = tab.length) > 0) {
                int index = (n - 1) & root.hash;
                TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
                if (root != first) {
                    Node<K,V> rn;
                    tab[index] = root;
                    TreeNode<K,V> rp = root.prev;
                    if ((rn = root.next) != null)
                        ((TreeNode<K,V>)rn).prev = rp;
                    if (rp != null)
                        rp.next = rn;
                    if (first != null)
                        first.prev = root;
                    root.next = first;
                    root.prev = null;
                }
                assert checkInvariants(root);
            }
        }

        /**
         * Finds the node starting at root p with the given hash and key.
         * The kc argument caches comparableClassFor(key) upon first use
         * comparing keys.
         */
        final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
            TreeNode<K,V> p = this;
            do {
                int ph, dir; K pk;
                TreeNode<K,V> pl = p.left, pr = p.right, q;
                if ((ph = p.hash) > h)
                    p = pl;
                else if (ph < h)
                    p = pr;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                else if (pl == null)
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.find(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
            return null;
        }

        /**
         * Calls find for root node.
         */
        final TreeNode<K,V> getTreeNode(int h, Object k) {
            return ((parent != null) ? root() : this).find(h, k, null);
        }

        /**
         * Tie-breaking utility for ordering insertions when equal
         * hashCodes and non-comparable. We don't require a total
         * order, just a consistent insertion rule to maintain
         * equivalence across rebalancings. Tie-breaking further than
         * necessary simplifies testing a bit.
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
         * Forms tree of the nodes linked from this node.
         * @return root of tree
         */
        final void treeify(Node<K,V>[] tab) {
            TreeNode<K,V> root = null;
            for (TreeNode<K,V> x = this, next; x != null; x = next) {
                next = (TreeNode<K,V>)x.next;
                x.left = x.right = null;
                if (root == null) {
                    x.parent = null;
                    x.red = false;
                    root = x;
                }
                else {
                    K k = x.key;
                    int h = x.hash;
                    Class<?> kc = null;
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
            moveRootToFront(tab, root);
        }

        /**
         * Returns a list of non-TreeNodes replacing those linked from
         * this node.
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
         * Tree version of putVal.
         */
        final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            TreeNode<K,V> root = (parent != null) ? root() : this;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);
                }

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    Node<K,V> xpn = xp.next;
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    moveRootToFront(tab, balanceInsertion(root, x));
                    return null;
                }
            }
        }

        /**
         * Removes the given node, that must be present before this call.
         * This is messier than typical red-black deletion code because we
         * cannot swap the contents of an interior node with a leaf
         * successor that is pinned by "next" pointers that are accessible
         * independently during traversal. So instead we swap the tree
         * linkages. If the current tree appears to have too few nodes,
         * the bin is converted back to a plain bin. (The test triggers
         * somewhere between 2 and 6 nodes, depending on tree structure).
         */
        final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable) {
            int n;
            if (tab == null || (n = tab.length) == 0)
                return;
            int index = (n - 1) & hash;
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
            if (pred == null)
                tab[index] = first = succ;
            else
                pred.next = succ;
            if (succ != null)
                succ.prev = pred;
            if (first == null)
                return;
            if (root.parent != null)
                root = root.root();
            if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                tab[index] = first.untreeify(map);  // too small
                return;
            }
            TreeNode<K,V> p = this, pl = left, pr = right, replacement;
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                while ((sl = s.left) != null) // find successor
                    s = sl;
                boolean c = s.red; s.red = p.red; p.red = c; // swap colors
                TreeNode<K,V> sr = s.right;
                TreeNode<K,V> pp = p.parent;
                if (s == pr) { // p was s's direct parent
                    p.parent = s;
                    s.right = p;
                }
                else {
                    TreeNode<K,V> sp = s.parent;
                    if ((p.parent = sp) != null) {
                        if (s == sp.left)
                            sp.left = p;
                        else
                            sp.right = p;
                    }
                    if ((s.right = pr) != null)
                        pr.parent = s;
                }
                p.left = null;
                if ((p.right = sr) != null)
                    sr.parent = p;
                if ((s.left = pl) != null)
                    pl.parent = s;
                if ((s.parent = pp) == null)
                    root = s;
                else if (p == pp.left)
                    pp.left = s;
                else
                    pp.right = s;
                if (sr != null)
                    replacement = sr;
                else
                    replacement = p;
            }
            else if (pl != null)
                replacement = pl;
            else if (pr != null)
                replacement = pr;
            else
                replacement = p;
            if (replacement != p) {
                TreeNode<K,V> pp = replacement.parent = p.parent;
                if (pp == null)
                    root = replacement;
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement;
                p.left = p.right = p.parent = null;
            }

            TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

            if (replacement == p) {  // detach
                TreeNode<K,V> pp = p.parent;
                p.parent = null;
                if (pp != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                }
            }
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

        /* ------------------------------------------------------------ */
        // Red-black tree methods, all adapted from CLR

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

        static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
            x.red = true;
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;
                if (xp == (xppl = xpp.left)) {
                    if ((xppr = xpp.right) != null && xppr.red) {
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.right) {
                            root = rotateLeft(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                else {
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }

        static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                                   TreeNode<K,V> x) {
            for (TreeNode<K,V> xp, xpl, xpr;;)  {
                if (x == null || x == root)
                    return root;
                else if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                else if (x.red) {
                    x.red = false;
                    return root;
                }
                else if ((xpl = xp.left) == x) {
                    if ((xpr = xp.right) != null && xpr.red) {
                        xpr.red = false;
                        xp.red = true;
                        root = rotateLeft(root, xp);
                        xpr = (xp = x.parent) == null ? null : xp.right;
                    }
                    if (xpr == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                        if ((sr == null || !sr.red) &&
                            (sl == null || !sl.red)) {
                            xpr.red = true;
                            x = xp;
                        }
                        else {
                            if (sr == null || !sr.red) {
                                if (sl != null)
                                    sl.red = false;
                                xpr.red = true;
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
                else { // symmetric
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
         * Recursive invariant check
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