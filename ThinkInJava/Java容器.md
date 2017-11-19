# Java容器:

> 本文只记录关键知识点,API自己看文档

## List:

List承诺可以将元素维护在特定的序列当中,它在Collection的基础上添加了大量方法,使得可以在List中插入和移除元素.List的行为依据`equals()`的行为所变化.

+ 有两种类型的List:

    1. ArrayList: 底层实现基于数组,擅长随机访问元素,但是在List中间插入移除元素代价比较大.

    2. LinkedList: 底层实现基于双向链表,在List中插入删除和随机访问代价较低,随机访问较为逊色.

        + `LinkedList`中还添加了可以使其用作栈,队列或双端队列的方法:

            1. `element()`, `getFirst()`和`peek()`都返回列表的头一个元素.如果List为空,则抛出`NoSuchElementException`.而`peek()`返回`null`.
            2. `removeFirst()`,`remove()`和`poll()`移除并放回列表的头元素.如果为空,抛出`NNoSuchElementException`,而`poll()`返回`null`.`

## Set

Set用于保存不重复的元素,元素必须定义`equals()`方法来确保对象的唯一性.

+ 三种类型的Set:

    1. HashSet: 使用了散列函数,为快速查找而设计的Set,存入的元素必须定义`hashCode()`.

    2. TreeSet: 保持次序的`Set`,底层是`红-黑树`.使用它可以从`Set`中提取有序的序列.元素必须实现`Comparable`接口.(可传入比较器或者覆盖`compareTo()`自定义排序).

    3. LinkedHashSet: 具有`HashSet`的查询速度,且内部使用链表维护元素(插入)顺序,迭代遍历时,结果会按插入次序显示.同样元素要实现`hashCode()`.

+ BitSet:

    高效地存储大量"开/关"信息.效率是指空间,访问时间比数组慢.


## Map

Map可以将对象映射到其他对象进行存储.元素需要实现`equals()`

+ 六种类型的Map:

    1. HashMap: 基于散列表实现(取代了Hashtable).插入和查询时"键值对"的开销是固定的.可以通过构造器设置`容量`和`负载因子`,以调整容器的性能.
        + `HashMap`使用了特殊的值,称作`散列码`,用来取代对键的缓慢搜索,`散列码`是"相对唯一"的,用来代表对象的int值,它通过对象的某些信息换算而成.

    2. LinkedHashMap: 类似于`HashMap`但是迭代遍历时,取得的"键值对"是其插入次序,或者是最近最少使用次序(LRU).比`HashMap`慢一点,但迭代访问速度更快一点.底层用到了链表.

    3. TreeMap: 基于红黑数实现,查看"键"或"键值对"时,它们会被排序(次序由`Comparable`或`Comparator`决定).遍历所得到的结果是经过排序的.可以通过`subMap()`返回一个子树.

    4. WeakHashMap: 使用`弱键`映射,允许虚拟机自动清理键和值.如果除映射之外没有引用指向这个"键",则这个"键"会被回收.

    5. ConcurrentHashMap: 一种线程安全的Map,它不涉及同步加锁.

    6. IdentityHashMap: 使用`==`代替`equals()`对键进行比较的散列映射.

+ 散列与散列码:

    1. 如果要用自己的类作为HashMap的键,就必须同时覆盖`hashCode()`和`equals()`.<br>
    2. `hashCode()`不需要总能返回唯一的标识码,但`equals()`必须严格判断两个对象异同.<br>
    3. `hashCode()`用来查找对象,`equals()`用来严格判断对象是否与表中的键相同.<br>

    + `equals()`必须满足5个条件:

        1. 子反性:　对任意ｘ，ｘ.equals(x)一定返回true.
        2. 对称性: 对任意ｘ和ｙ, 如果ｘ.equals(y)返回true,y.equals(x)也一定返回true.
        3. 传递性: 对任意x,y,z,如果有x.equals(y)和y.equals(x)返回true.则x.equals(z)返回true.
        4. 一致性: 任意x.equals(y)要是幂等的.
        5. 任意不是null的x,x.equals(null)一定是false.

+ 散列的实现:

    1. Map中用数组来保存键的信息.

    2. 通过键对象的`hashCode()(散列函数)`生成`散列码`,作为数组的下标.

    3. 数组中保存拥有散列码和该位置下标相同的"值"的`List`,查询时就通过`equals()`方法对`List`中的值进行线性查询.

    > 注: 新版的jdk是基于List和红黑数实现(数据少时用list,大时用树)


## Queue

可以将元素从队列的一段插入,并于另一端将他们抽取出来.

+ 在JavaSE5中有两种实现(排除并发的队列);

    1. LinkedList.

    2. PriorityQueue: 优先队列,排序顺序通过实现`Comparable`而进行控制.

+ 双端队列:

    java中没有显示用于双向队列的接口,但是`LinkedList`包含支持双端队列的方法,可以通过组合来进行创建.



## Stack

一般通过`LinkedList`实现.

## 迭代器

迭代器是一个对象,它的工作是遍历并选择序列中的对象,而不必知道或者关心该序列底层的结构.任何实现`Iterable`接口的类,其对象都可以用`foreach`语句遍历.

+ Java的`Iterator`只能单向移动,它能用来:

    1. 使用方法`iterator()`要求容器返回一个`Iterator`,Iterator将准备好返回序列的第一个元素.

    2. 使用`next()`获得序列的下一个yuans.

    3. 使用`hasNext()`检查序列是否还有元素.

    4. 使用`remove()`将迭代器新返回的元素删除.

+ ListIterator:

    `ListIterator`是一个更加强大的`Iterator`子类型,当是只能用于`List`类型.

    `ListIterator`可以双向移动,可以产生当前位置的前一个和后一个索引,并且可以通过`set()`方法替换它访问过的最后一个元素.


## 线程安全的容器:

(有空补充...)