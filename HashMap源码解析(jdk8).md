# HashMap源码解析(jdk8版)

HashMap允许key为null,也允许value为null<br/>
HashMap跟HashTable两个类是差不多的,除了HashTable是线程安全且不允许null值这一点外.

基本概念:HashMap底层是数组+链表(数组的每个值都是一条链表的头结点),1.8后加入了红黑树(当链表长度达到8就自动将该链表替换为红黑树),通过计算key的哈希码,在经过高位参与位运算计算得出键值对(将key和value包装起来的对象)所在的数组的下标,采用头插入法插入该位置的链表(若该位置是空的就直接插入)

## HashMap的相关域:


        static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

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
            return (e = getNode(hash(key), key)) == null ? null : e.value;      //1
        }

        final Node<K,V> getNode(int hash, Object key) {
            Node<K,V>[] tab;
            Node<K,V> first, e;
            int n;
            K k;
            if ((tab = table) != null && (n = tab.length) > 0 &&
                (first = tab[(n - 1) & hash]) != null) {                        //2
                if (first.hash == hash && // always check first node
                    ((k = first.key) == key || (key != null && key.equals(k)))) //3
                    return first;
                if ((e = first.next) != null) {                                 //4
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

## 增加键值对`put(K key, V value)`

        public V put(K key, V value) {
            return putVal(hash(key), key, value, false, true);
        }

        /**
        * Implements Map.put and related methods
        *
        * @param hash hash for key
        * @param key the key
        * @param value the value to put
        * @param onlyIfAbsent if true, don't change existing value
        * @param evict if false, the table is in creation mode.
        * @return previous value, or null if none
        */
        final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                    boolean evict) {
            Node<K,V>[] tab;
            Node<K,V> p;
            int n, i;
            if ((tab = table) == null || (n = tab.length) == 0)             //1
                n = (tab = resize()).length;
            if ((p = tab[i = (n - 1) & hash]) == null)                      //2
                tab[i] = newNode(hash, key, value, null);
            else {
                Node<K,V> e; K k;
                if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k)))) //3
                    e = p;
                else if (p instanceof TreeNode)                             //4
                    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
                else {                                                      //5
                    for (int binCount = 0; ; ++binCount) {
                        if ((e = p.next) == null) {                         
                            p.next = newNode(hash, key, value, null);
                            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                                treeifyBin(tab, hash);
                            break;
                        }
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                            break;
                        p = e;
                    }
                }
                if (e != null) { // existing mapping for key                //6
                    V oldValue = e.value;
                    if (!onlyIfAbsent || oldValue == null)
                        e.value = value;
                    afterNodeAccess(e);
                    return oldValue;
                }
            }                                                               //7
            ++modCount;
            if (++size > threshold)
                resize();
            afterNodeInsertion(evict);
            return null;
        }

1. 如果数组为null或是空的,则`resize()`扩充容量
2. 通过hash计算并位运算取摸获得数组下标,若该位置是空的,新建链表结点直接填坑然后跳到7,否则进入3
3. 判断头结点的key跟要put进去的key是否同一个,是则将其引用赋给e,进入6,否则进入4
4. 判断头结点是不是树结点,是则执行`putTreeVal`,若树中已存在该key,则直接返回该键值对(赋给e),否则新建并插入结点并返回null,然后进入6.如果不是树节点则进入5
5. 在链表中遍历,如果不存在,就新建一个结点,然后是否达到树化的阙值,是就转化为树结构,之后跳到7.如果存在就把它的引用赋给e跳到6
6. 在搜索到当前map中存在相同key时候将该键值对赋给e,在这里进行值的覆盖,并返回旧值
7. 对改动进行计数,判断是否需要进行数组扩容,返回null


## 移除键值对`remove(Object key)`

        public V remove(Object key) {
            Node<K,V> e;
            return (e = removeNode(hash(key), key, null, false, true)) == null ?
                null : e.value;
        }

        final Node<K,V> removeNode(int hash, Object key, Object value,
                                boolean matchValue, boolean movable) {
            Node<K,V>[] tab;
            Node<K,V> p;
            int n,index;
            if ((tab = table) != null && (n = tab.length) > 0 &&            //1
                (p = tab[index = (n - 1) & hash]) != null) {
                Node<K,V> node = null, e;
                K k;
                V v;
                if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k)))) //2
                    node = p;
                else if ((e = p.next) != null) {                            //3
                    if (p instanceof TreeNode)
                        node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                    else {
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
                if (node != null && (!matchValue || (v = node.value) == value ||    //4
                                    (value != null && value.equals(v)))) {
                    if (node instanceof TreeNode)
                        ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                    else if (node == p)
                        tab[index] = node.next;
                    else
                        p.next = node.next;
                    ++modCount;
                    --size;
                    afterNodeRemoval(node);
                    return node;
                }
            }
            return null;
        }

1. 判断底层数组是否为null或者是空的,是就直接返回null,否则2
2. 判断头结点是否就是要移除的键值对,是就赋给e,进入4,否则进入3
3. 判断是树还是链表并进行相应遍历,找到符合的键值对,并赋给e,进入4,若查无,返回null
4. 针对不同的存储结构进行相应的移除操作,并更新相关的计数值

## 链表的树化操作`treeifyBin(Node<K,V>[] tab, int hash)`

        /**
        * Replaces all linked nodes in bin at index for given hash unless
        * table is too small, in which case resizes instead.
        */
        final void treeifyBin(Node<K,V>[] tab, int hash) {
            int n, index; Node<K,V> e;
            if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
                resize();
            else if ((e = tab[index = (n - 1) & hash]) != null) {
                TreeNode<K,V> hd = null, tl = null;
                do {
                    TreeNode<K,V> p = replacementTreeNode(e, null);
                    if (tl == null)
                        hd = p;
                    else {
                        p.prev = tl;
                        tl.next = p;
                    }
                    tl = p;
                } while ((e = e.next) != null);
                if ((tab[index] = hd) != null)
                    hd.treeify(tab);
            }
        }

+ 先将链表结点转化成树结点,构造成双向链表,在`treeify`进行红黑树的构造

## 扩容操作`resize`

        /**
        * Initializes or doubles table size.  If null, allocates in
        * accord with initial capacity target held in field threshold.
        * Otherwise, because we are using power-of-two expansion, the
        * elements from each bin must either stay at same index, or move
        * with a power of two offset in the new table.
        *
        * @return the table
        */
        final Node<K,V>[] resize() {
            Node<K,V>[] oldTab = table;
            int oldCap = (oldTab == null) ? 0 : oldTab.length;
            int oldThr = threshold;
            int newCap, newThr = 0;
            if (oldCap > 0) {
                if (oldCap >= MAXIMUM_CAPACITY) {                           // 1
                    threshold = Integer.MAX_VALUE;
                    return oldTab;
                }
                else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&       // 2
                        oldCap >= DEFAULT_INITIAL_CAPACITY)
                    newThr = oldThr << 1; // double threshold
            }
            else if (oldThr > 0) // initial capacity was placed in threshold    // 3
                newCap = oldThr;
            else {               // zero initial threshold signifies using defaults // 4
                newCap = DEFAULT_INITIAL_CAPACITY;
                newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
            }
            if (newThr == 0) {                                                      // 5
                float ft = (float)newCap * loadFactor;
                newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                        (int)ft : Integer.MAX_VALUE);
            }
            threshold = newThr;                                                     // 6
            @SuppressWarnings({"rawtypes","unchecked"})
                Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
            table = newTab;
            if (oldTab != null) {                                                   // 7
                for (int j = 0; j < oldCap; ++j) {
                    Node<K,V> e;
                    if ((e = oldTab[j]) != null) {
                        oldTab[j] = null;
                        if (e.next == null)                                         // 7-1
                            newTab[e.hash & (newCap - 1)] = e;
                        else if (e instanceof TreeNode)                             // 7-2
                            ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                        else { // preserve order                                    // 7-3
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

1. 若底层数组长度大于等于允许的最大值,将扩容阙值设为MAX_INT,直接不做任何操作,直接返回原数组
2. 如果底层数组长度是大于默认初始长度且当前长度*2小于允许的最大值,则将新的数组长度,扩容阙值都设为原来的两倍
3. 当前数组未初始化,且扩容阙值已经初始化(不为0),将新的数组长度设定为扩容阙值,跳到5
4. 当前数组与扩容阙值都未初始化,将新的数组长度和扩容阙值设为默认初始值
5. 根据新的数组长度值计算新的扩容阙值,如果新的数组长度值或者新的阙值大于数组长度的允许最大值,则将其替换为MAX_INT,反之保留
6. 将经过上述计算得到的新值进行更新(设置threshold为新值, 实例化一个新长度的底层数组)
7. 遍历数组的每个坑位,将老数组的值搬运到新的数组中
7-1. 若该坑位只有一个结点,直接搬运到新数组对应坑位,需要重新计算下标,因为新数组的长度已经改变
7-2. 若该坑位放的是树,则调用对应方法进行换坑
7-3. 若该坑位是是链表,遍历这条链表,根据其hash&旧数组长度是0还是1分为两组,一组在新数组下标不变,另一组是原来下标+旧数组长度<br/>
注: 因为每次扩容都是2扩容两倍,位运算时只增加一个高位(右数第oldCap个),按位与时,若键值对的右数第oldCap位是0则下标不会受扩容影响,若不是,则下标是原下标加上oldCap.


> 以上分析为个人理解,欢迎指正!


关于红黑树的实现与操作并没有深入代码层次解析，有兴趣可阅读［］（https://tech.meituan.com/redblack-tree.html）