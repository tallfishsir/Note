##### Q1：HashMap 常见成员变量和常量及含义？

```java
/**
 * capacity 默认初始值 16，且 capacity 的值只能是 2 的次幂
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
/**
 * capacity 的最大值
 */
static final int MAXIMUM_CAPACITY = 1 << 30;
/**
 * 默认加载因子数值，当元素个数所占容量超过加载因子比例，就需要扩容
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
/**
 * 链表转为红黑树的分界值
 */
static final int TREEIFY_THRESHOLD = 8;
/**
 * 红黑树转为链表的分界值
 */
static final int UNTREEIFY_THRESHOLD = 6;
/**
 * 链表转红黑树时，所需的最小元素数量
 */
static final int MIN_TREEIFY_CAPACITY = 64;

/**
 * 存放所有元素的 Node 数组
 */
transient Node<K,V>[] table;
/**
 * HashMap 键值对的 Set 集合
 */
transient Set<Map.Entry<K,V>> entrySet;
/**
 * HashMap 存储的元素个数
 */
transient int size;
/**
 * 加载因子
 */
final float loadFactor;
/**
 * 下一次需要扩容的值 (capacity * loadFactor).
 */
int threshold;
```

##### Q2：HashMap 通过不同构造函数创建后的初始状态？

HashMap 底层基于哈希表，采用数组存放数组，使用链表来解决哈希碰撞，还引入红黑树解决链表长度过长导致的查询速度下降问题。它有三个常见构造函数

```java
/**
 * 空参数的构造函数，加载因子设置为默认值 0.75
 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; 
}

/**
 * 设置初始容量，加载因子设置为默认值 0.75
 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
 * 设置加载因子，并根据初始容量，设置下一次扩容的值
 */
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

/**
 * 返回一个比入参大的最近的 2 的次冥
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

空入参的构造函数，只设置了加载因子 loadFactor 为默认值，这时候 table 为 null， threshold 为 0。

有两个入参的构造函数会对入参先进行校验，如果正常，会设置 加载因子 loadFactor 的值，然后通过 tableSizeFor() 方法计算 threshold 的值。这时候 table 为 null，threshold 为 2次幂。

##### Q3：HashMap 为什么容量必须是 2 的次冥，又是如何保证的？

HashMap 存放元素的时候，都是根据 key 来确定元素在 table数组中的位置的，那么这个计算地址的方法是否高效就影响了 HashMap 的存取效率。

首先，需要计算 key 的hash 值，为了减少碰撞率，让 hash 值的低16位也参与影响了最后的返回值。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

index = (length - 1) & hash
```

然后确定该元素在 table 数组的位置就用到了「容量必须是 2 的次冥」这个性质。将数组长度减一，就会变成「11..11」再和元素 hash 值相与取模，就可以得到一个数组中的位置。

创建 HashMap 时，tableSizeFor 就是用来保证数组长度一定是  2 次幂。

```java
// 2 次幂的特点是二进制标识的时候，只有一位是 1，剩下的都是 0，那么我们要取一个数最近的 2 次幂，就把这个数二进制最左边的1的左边位置设置 1 ，然后剩下的位置全部设置 0。
// 我们都知道 >>> 是无符号右移，左边的位置补 0 
// 为了防止输入的值 就是 2 次幂，先对入参减一
int n = cap - 1;
// 从 >>>1 到 >>>16 保证了一共 32 位的 int 值，左边全部设置为 1
n |= n >>> 1;
n |= n >>> 2;
n |= n >>> 4;
n |= n >>> 8;
n |= n >>> 16;
// 在 ...111的基础上 +1，就会变成 ...1000，变成 2次幂
return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
```

##### Q4：HashMap 添加元素源码分析和扩容机制？

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    // 存放元素 table 数组的局部变量
    Node<K,V>[] tab; 
    // table 数组某个位置的元素
    Node<K,V> p; 
    // table 的容量 capacity
    int n;
    // 某个元素通过 (capacity - 1) & hash 计算出在数组中的位置
    int i;
    // 如果 table 是空值，需要通过 resize() 方法来初始化，然后再设置 n 为当前数组的容量
    if ((tab = table) == null || (n = tab.length) == 0)
        // resize()不仅用来调整大小，还用来进行初始化配置
        n = (tab = resize()).length;
    // 如果 hash % (capacity - 1) 位置没有值，就把元素直接添加进去
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果 hash % (capacity - 1) 位置有值，才会进行可能的添加链表、树化等
    else {
        // table 数组某个位置已经存放的元素
        Node<K,V> e;
        // Node 键值
        K k;
        // 如果插入元素的 hash 和 key 与已经存在元素的 hash、key 完全相同，e 就会设置为一个重复元素标志
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 进入插入元素的流程，先判断当前位置上 hash 碰撞所形成的链表是否已经转为红黑树，如果是红黑树就完成树的操作
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 如果不是树，就顺着链表向后查找
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 找到链表结尾，并插入，binCount ＞ 8 了，那说明链表长度也超过了转为树的分界值 8
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        // 将 table 某个位置上的链表转为红黑树
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果插入元素的 hash 和 key 与链表上已存在元素的 hash、key 完全相同，就退出循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果插入元素的 hash 和 key 与链表上已存在元素的 hash、key 完全相同，替换
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 无论前面是直接插入，还是添加链表或者树化，这里 modCount 都会自增1
    ++modCount;
    // 对于替换元素的情况，前面if (e != null) 的分支已经处理，
    // 这里集合中保存的元素数量都已经增加了一个，如果增加后到达需要扩容的临界值，需要扩容。
    if (++size > threshold)
        resize();
    // 默认也是空实现，允许我们插入完成后做一些操作
    afterNodeInsertion(evict);
    return null;
}
```

在 putVal() 方法中，使用的树化的方法是 treeifyBin

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    // table 的容量 capacity
    int n;
    // 通过 hash 计算出来在 table 中的位置
    int index;
    // table 数组 index 位置的元素
    Node<K,V> e;
    // 如果 table 元素为空或者 table 长度没有达到建立树的要求 64，就重新计算容量
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // 树的头结点
        TreeNode<K,V> hd = null, 
        // 当前结点的前一个结点
        TreeNode<K,V> tl = null;
        do {
             // 将当前元素 Node 转为 TreeNode 结点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            // 设置头结点
            if (tl == null)
                hd = p;
            else {
                // 设置结点的双向
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        // 链表数据的顺序是不符合红黑树的，所以需要调整
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

无论是在`put`还是`treeify`时，都依赖于`resize`，它的重要性不言而喻。它不仅可以调整大小，还能调整树化和反树化（从树变为链表）所带来的影响。

resize 一种调用情况是在空参数构造函数创建 HashMap 后，添加第一个元素。这时 table == null size == 0 threshold == 0 loadFractor = 0.75，然后设置 newCap 为默认容量 16，扩容临界值 newThr = 16*0.75。

另一种是在指定容量大小的构造函数创建 HashMap 后，添加第一个元素，这时 table  == null  size == 0 threshold == 2次幂  loadFractor = 0.75，然后就把初始化时的扩容临界值 threshold 设置 newCap，此时 newThr = 0，后面会设置newThr 为 newCap*0.75

还有一种情况就是在添加元素时，超过了扩容分界值，这时 table  != null  size == 某个值 threshold ==2次冥  loadFractor = 0.75，如果扩容前容量 size 超过 2^30，就把扩容分界值设置为 Integer.MAX_VALUE，如果没有超过就扩容两倍

```java
final Node<K,V>[] resize() {
    // 扩容前的 table 数组
    Node<K,V>[] oldTab = table;
    // 扩容前的 table 数组长度size，空数组长度为 0
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
     // 扩容前需要扩容的临界值
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 针对非 初始化后第一次 添加元素
    if (oldCap > 0) {
        // 扩容前容量 size 超过 2^30，就把扩容分界值设置为 Integer.MAX_VALUE
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容前容量 size 超过默认容量 16，设置 newCap 是 newCap 的2倍，newThr 是 oldThr 的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    else if (oldThr > 0) 
         // 初始化后第一次添加元素，且初始化是设置扩容临界值的，就把初始化时的扩容临界值 threshold 设置 newCap，此时 newThr = 0
        newCap = oldThr;
    else {
        // 初始化后第一次添加元素，且初始化是空构造函数，就设置 newCap 为默认容量 16，扩容临界值 newThr = 16*0.75
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 初始化后第一次添加元素，且初始化是设置扩容临界值的，设置 newThr = newCap*loadFactor
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    // 创建一个 newCap 长度的新数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 数据拷贝
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 只有一个值，重新计算位置
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 重新规划树，如果树的size很小，默认为6，就退化为链表
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {
                    // 处理链表的数据
                    // loXXX 指的是在原表中出现的位置
                    Node<K,V> loHead = null, loTail = null;
                    // hiXXX 指的是在原表中不包含的位置
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 这里把hash值与oldCap按位与。
                        // oldCap是2的次幂，所以除了最高位为1以外其他位都是0
                        // 和它按位与的结果为0，说明hash比它小，原表有这个位置
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
```
