# ConcurrentHashMap源码分析（基于jdk1.8）

# 简介

1.8版本的并发HashMap相较与1.7的版本，去掉了分段锁，每次操作都在hash table的一个桶位上进行cas或加锁操作，细化了加锁粒度，提高并发能力。初次之外，在哈希冲突处理方面更新了增到8转红黑树，减到6转链表。



 We do not want to waste the space required to associate a distinct lock object with each bin, so instead use the first node of a bin list itself as a lock. Locking support for these locks relies on builtin "synchronized" monitors.

 The main disadvantage of per-bin locks is that other update operations on other nodes in a bin list protected by the same lock can stall

 public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    }

### 类属性

+ private static final int MAXIMUM_CAPACITY = 1 << 30; 

    table桶数最大值，前两位用作控制标志


+ private static final int DEFAULT_CAPACITY = 16;

    table桶数初始化默认值，需为2的幂次方
    /**
     * The largest possible (non-power of two) array size.
     * Needed by toArray and related methods.
     */
+ static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;


+ private static final float LOAD_FACTOR = 0.75f;

    加载因子，扩容的阀值，可在构造方法定义


+ static final int TREEIFY_THRESHOLD = 8;

    树化阀值1，当链表节点超过8允许转化为红黑树

+ static final int UNTREEIFY_THRESHOLD = 6;

    链化阀值1，当树节点小于6则转化为链表

+ static final int MIN_TREEIFY_CAPACITY = 64;

    树化阀值2，当数组桶树达到64以上才允许链表树化

+ private static final int MIN_TRANSFER_STRIDE = 16;

    链化阀值2，与上类似

    /**
     * The number of bits used for generation stamp in sizeCtl.
     * Must be at least 6 for 32bit arrays.
     */
    private static int RESIZE_STAMP_BITS = 16;

    /**
     * The maximum number of threads that can help resize.
     * Must fit in 32 - RESIZE_STAMP_BITS bits.
     */
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

    /**
     * The bit shift for recording size stamp in sizeCtl.
     */
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

    /*
     * Encodings for Node hash fields. See above for explanation.
     */
    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

    /** Number of CPUS, to place bounds on some sizings */
    static final int NCPU = Runtime.getRuntime().availableProcessors();


## 主要的内部类

### Node

+ final保证线程安全性 volatile保证线程间修改（update）的可见性

    static class Node<K,V> implements Map.Entry<K,V> {
        // 键值对的hash计算值
        final int hash;
        final K key;
        volatile V val;
        // 下一个节点，链表法解决冲突
        volatile Node<K,V> next;
    }

## 对象属性

+ transient volatile Node<K,V>[] table;

    桶数组，数组长度为2的幂次方

+ private transient volatile Node<K,V>[] nextTable;

    resize操作使用的桶数组

    /**
     * Base counter value, used mainly when there is no contention,
     * but also as a fallback during table initialization
     * races. Updated via CAS.
     */
    private transient volatile long baseCount;

    /**
     * Table initialization and resizing control.  When negative, the
     * table is being initialized or resized: -1 for initialization,
     * else -(1 + the number of active resizing threads).  Otherwise,
     * when table is null, holds the initial table size to use upon
     * creation, or 0 for default. After initialization, holds the
     * next element count value upon which to resize the table.
     */
    private transient volatile int sizeCtl;

    /**
     * The next table index (plus one) to split while resizing.
     */
    private transient volatile int transferIndex;

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
     */
    private transient volatile int cellsBusy;

    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    private transient volatile CounterCell[] counterCells;

+ private transient KeySetView<K,V> keySet;<br/>private transient ValuesView<K,V> values;<br/>private transient EntrySetView<K,V> entrySet;

    数据视图


## 主要的几个方法

### spread

再散列，通过哈希吗的高位与低位抑或，使K-V的分布更均匀。HASH_BITS为7fffffff,通过与其相与保障最高位为0，避免出现ffff0000^0000ffff=ffffffff发送数组越界的情况

    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }


### get
    
    // 与HashMap不同，key不能为Null
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        // spread通过高位参与运算，获取数组的桶位
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            // 取出该桶位的第一个节点
            (e = tabAt(tab, (n - 1) & h)) != null) {
            // 若第一个节点就是当前key，直接返回第一个节点的val
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // 在冲突解决的数据结构中查找节点并返回其val
            // eh<0 --> hash最高位为1，代表当前桶位冲突解决使用的是红黑树，find查找
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // eh<0 --> hash最高位为0，代表当前桶位冲突解决使用的是链表，直接遍历查询
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }

### put

put实际实现为putVal，增加一个参数，判断是否覆盖原有的K-V

Key和Value都不能为Null

    /**
     * Maps the specified key to the specified value in this table.
     * Neither the key nor the value can be null.
     *
     * <p>The value can be retrieved by calling the {@code get} method
     * with a key that is equal to the original key.
     */
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 若桶数组仍未初始化，则先初始化数组
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 若该桶位无头节点，则通过CAS操作插入桶位，退出循环
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 查看该头节点状态，若为MOVED（当前正在resize操作），执行helpTransfer帮助resize
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            // 当前头节点已经存在或者CAS操作失败，对头结点synchronized加对象锁，执行冲突解决
            else {
                V oldVal = null;
                synchronized (f) {
                    // 双重检查
                    if (tabAt(tab, i) == f) {
                        // 若是链表
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    // 发现插入Key已存在，根据onlyIfAbsent执行覆盖
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 没有相同的Key,接入链表尾部
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 若是红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // bitCount为链插入后的长度,达到阀值就执行树化操作
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // K-V计数加一
        addCount(1L, binCount);
        return null;
    }

### size

size操作返回的值来自于CounterCell[]中元素值之和，其中的元素改动又来自于addCount方法

可知CounterCell[]其中每个元素代表着桶数组中每个元素内的节点数

    /**
     * {@inheritDoc}
     */
    public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }

     @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }

    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }

### resize

