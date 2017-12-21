# HashMap源码解析(jdk8版)

HashMap允许key为null,也允许value为null<br/>
HashMap跟HashTable两个类是差不多的,除了HashTable是线程安全且不允许null值这一点外.

基本概念:HashMap底层是数组+链表(数组的每个值都是一条链表的头结点),1.8后加入了红黑树(当链表长度达到8就自动将该链表替换为红黑树),通过计算key的哈希码,在经过高位参与位运算计算得出键值对(将key和value包装起来的对象)所在的数组的下标,采用头插入法插入该位置的链表(若该位置是空的就直接插入)

## HashMap的相关域:


    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    static final int MAXIMUM_CAPACITY = 1 << 30;

    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    static final int TREEIFY_THRESHOLD = 8;

    static final int UNTREEIFY_THRESHOLD = 6;

    static final int MIN_TREEIFY_CAPACITY = 64;

+ DEFAULT_INITIAL_CAPACITY: 默认底层数组的初始大小(2^4),可通过构造参数指定
+ MAXIMUM_CAPACITY: 数组的最大长度(2^30),超过将替换为此值
+ DEFAULT_LOAD_FACTOR: 默认负载因子,为0.75,可通过构造参数指定
+ TREEIFY_THRESHOLD: 链表转换为红黑树的阙值,当链表长度达到8自动转为红黑树进行存储(前提是数组长度大于等于64)
+ UNTREEIFY_THRESHOLD: 红黑树转为链表的阙值,当红黑树结点个数减小到6时,自动转为链表存储
+ MIN_TREEIFY_CAPACITY: 链表进行树化的前提条件,数组长度要达到64或一上,在这之前只能通过数组扩容来减少链表长度


    transient Node<K,V>[] table;

    transient Set<Map.Entry<K,V>> entrySet;

    transient int size;

    transient int modCount;

    int threshold;

    final float loadFactor;

+ table: 底层数组,可动态扩容,数组长度为2的整数次方
+ size: 这个map中存放的键值对数目
+ modCount: 记录这个map数据结构发生改变的次数(发送插入删除或者链表与树相互转换的操作),由于fail-fast机制
+ threshold: 数组进行扩容的下个阙值(当前键值对数量达到这个值后进行`resize()`(扩容)操作)(threshold = capacity * load factor)
+ loadFactor:　实际的负载因子



## 哈希码计算方法`hash(Object key)`

    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

+ 局部变量h存放hashCode()放回的初始哈希码,通过h右移16位与h异或(右移后,前16位为0,异或不改变h的前16位值)得到最终的哈希吗.<br/>
通过高位参与位运算可以减少数组长度较低时的哈希码冲突问题(取模时,高位变低位不变,冲突几率会变高)


## 链表结点的数据结构`Node<K,V>`

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; // 计算得到的hash码
        final K key;    // 键对象
        V value;        // 值对象
        Node<K,V> next; // 下一个结点
        方法略...
    }

## 树节点数据结构v`TreeNode<K,V>`

    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // 父亲节点
        TreeNode<K,V> left;    // 左子树
        TreeNode<K,V> right;   //右子树
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;            // 红色还是黑色
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        方法略...    
    }

+ 通过继承`LinkedHashMap.Entry<K,V>`,实际上间接继承了链表的`Node<K,V>`

## 获取value:get(Object key)

    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab;
        Node<K,V> first, e;
        int n;
        K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {        
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
1. 计算key的哈希码,传入getNode方法,放回Node对象或者null
2. 如果table为null,table是空的或者数组( (length-1)&hash )处的值为null,就返回null,否则进入3
3. 检查第一个结点,若是指定的key,直接返回该结点,否则进入4
4. 如果这个树/链表不止一个结点,先判断是树还是链表,再进行对应的结点查找,找到就返回,否则返回null.

未完待续...
