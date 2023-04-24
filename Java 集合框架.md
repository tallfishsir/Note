# Java集合框架

## ArrayList

List 接口定义的是一个允许重复的有序集合，具体实现 List 接口的实现类有：ArrayList、LinkedList、Vector：

- ArrayList：内部通过一个可以动态扩展大小的数组来存放元素
- LinkedList：内部通过链表来存放元素的，链表中的节点对象 Node 包含了数据值、前驱节点、后驱节点
- Vector：内部通过一个可以动态扩展大小的数组来存放元素，所有方法添加 synchronized 关键字修饰

ArrayList 内部是通过一个 Object 数组来保存数据的，初始默认大小是 10，当添加的元素数量超过数组长度时，会自动进行扩容，

```java
public class ArrayList<E> extends AbstractList<E>
		implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
	//ArrayList 容量的默认值
	private static final int DEFAULT_CAPACITY = 10;
	
	//elementData 中存放元素的上限
	private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
	
	//elementData 中实际存放的元素个数
	private int size;
	
	//空数组
	private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
	private static final Object[] EMPTY_ELEMENTDATA = {};
	
	//实际上存放元素的容器
	transient Object[] elementData; 
	
	public ArrayList() {
		this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
	}
	
	public ArrayList(int initialCapacity) {
		if (initialCapacity > 0) {
			this.elementData = new Object[initialCapacity];
		} else if (initialCapacity == 0) {
			this.elementData = EMPTY_ELEMENTDATA;
		} else {
			throw new IllegalArgumentException("Illegal Capacity: initialCapacity);
		}
	}
}
```

### 添加元素

ArrayList 添加元素时，默认是从队尾添加，整体流程是：

- 确认添加元素后，当前数据长度是否满足
- 如果数据长度还不满足，将数组长度扩容1.5倍
- 如果数据长度还不满足，直接将数组长度设置所需大小
- 然后判断数组长度如果超过了 MAX_ARRAY_SIZE，直接将数组长度设置 Integer.MAX_VALUE
- 最后将元素添加到 size++ 的位置

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;
    return true;
}

/**
 * 用于确认数组的大小，如果数组长度不满足所需要的容量，就会扩容
 * 如果 elementData 是一个空数组，取默认值和参数值的较大值，
 * 传入 ensureExplicitCapacity 确认最后的数组大小
 */
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}

/**
 * 确认是否需要扩容
 */
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // 如果需要的最小需求容量大于当前 elementData 的长度，就需要扩展数组长度到最小需求容量
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

/**
 * 扩容 elementData 的大小
 * 首先，对旧容量扩大至1.5倍，如果扩容1.5倍后还是无法满足最小需求容量，新容量就被赋值为最小需求容量
 * 如果扩容或者赋值最小需求容量 新容量比 MAX_ARRAY_SIZE 还大，就根据最小需求容量来确认最终 elementData 长度
 */
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}

/**
 * 在 扩容或者赋值最小需求容量后的新容量比 MAX_ARRAY_SIZE 还大的前提下
 * elementData 的长度根据最小需求容量来确定是 Integer.MAX_VALUE 还是 MAX_ARRAY_SIZE
 */
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

add(int index, E element)相较于add(E e) ，增加了入参检查和数组赋值操作 System.arraycopy()：

- 检查数组长度和扩容与 add() 相同
- System.arraycopy() 将 index 位置空出
- 在 index 位置插入元素

```java
/**
 * 在某个位置插入元素，需要把 index 及以后的元素往后移动一位
 */
public void add(int index, E element) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    elementData[index] = element;
    size++;
}
```

### 移除元素

ArrayList 移除元素底层是通过数组移动复制，来覆盖 index 位置上的元素，在复制移动后，还手动的将结尾多出来的元素设置为 null，从而让 GC 顺利完成内存清理。

```java
public E remove(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    modCount++;
    E oldValue = (E) elementData[index];
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}
```

在使用 for 循环遍历 ArrayList 的过程中，如果在循环体中调用 ArrayList 的 remove()`方法删除元素，ArrayList 会将被删除元素之后的所有元素向前移动一个位置，并将数组的长度减一。如果在循环体中删除元素，会导致后面的元素前移，使得当前正在访问的元素的下标发生改变，从而可能出现漏访问或重复访问的情况。

为了避免这种情况，可以使用迭代器来遍历 ArrayList，并使用迭代器的 remove() 方法删除元素。迭代器可以在遍历过程中删除元素，而不会影响到后续的遍历，因为迭代器内部维护了一个游标，可以动态调整位置。

```java
ArrayList<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
Iterator<Integer> iterator = list.iterator();
while (iterator.hasNext()) {
    Integer num = iterator.next();
    if (num % 2 == 0) {
        iterator.remove();
    } else {
        System.out.println(num);
    }
}
```

modCount 的作用主要体现在迭代器的使用上。每个迭代器都包含一个期望的 modCount 值，即迭代器创建时 ArrayList 的 modCount 值。在迭代器遍历 ArrayList 的过程中，如果发现 modCount 值与期望值不一致，就说明 ArrayList 在遍历过程中发生了结构性修改，迭代器就会抛出 ConcurrentModificationException 异常。

```java
private class Itr implements Iterator<E> {
	protected int limit = ArrayList.this.size;
	// 下一个要返回的 index
	int cursor;  
    // 上一个返回元素的 index
    int lastRet = -1;
    // 期望的修改次数
    int expectedModCount = modCount;
    
    //如果下一个返回元素的 index 小于总的元素个数，说明还有下一个
	public boolean hasNext() {
		return cursor < limit;
	}

	public E next() {
         // 实际修改值和期望修改值不一样，会抛异常
		if (modCount != expectedModCount)
			throw new ConcurrentModificationException();
         //从0开始遍历
		int i = cursor;
		if (i >= limit)
			throw new NoSuchElementException();
		Object[] elementData = ArrayList.this.elementData;
		if (i >= elementData.length)
			throw new ConcurrentModificationException();
         // 当返回值后，cursor 就指向了下一个元素的 index
		cursor = i + 1;
		return (E) elementData[lastRet = i];
	}
	
	public void remove() {
         // 上一个返回元素的 index<0 抛异常
		if (lastRet < 0)
			throw new IllegalStateException();
		if (modCount != expectedModCount)
			throw new ConcurrentModificationException();
		try {
             // 删除上一个返回的元素（涉及到数组的复制和移位，modCount修改）
			ArrayList.this.remove(lastRet);
             // 然后下一个index就指向被删除的元素的index，保证连续性
			cursor = lastRet;
			lastRet = -1;
             // 同步实际修改值和期望修改值
			expectedModCount = modCount;
             // 总个数减一
			limit--;
		} catch (IndexOutOfBoundsException ex) {
			throw new ConcurrentModificationException();
		}
	}
}
```

### System.arrayCopy/Arrays.copyOf 

System.arrayCopy() 是一个系统的原生方法，它的参数中需要一个目标数组，然后将原数组拷贝到目标数组中，可以选择拷贝的起点和长度。

Arrays.copyOf() 内部调用了System.arraycopy()  是方法内部新建一个数组，并返回该数组。

```java
public static native void arrayCopy(Object src, int srcPos, Object dest, int destPos, int length)

public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}

public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0, Math.min(original.length, newLength));
    return copy;
}
```

## LinkedList

Deque  接口是扩展 Queue 的双端队列，它支持在集合的两端插入和删除元素，实现 Deque 接口的实现类有 LinkedList：

- add()/addLast()/offer()/offerLast()：添加数据到队尾
- push()/addFirst()/offerFirst()：添加数据到队头
- removeFirst()/pollLast()：从队尾删除数据
- remove()/removeFirst()/pop()/poll()/pollFirst()：从队头删除数据

LinkedList 实现了 List 和 Qeque 两个接口，一般用它来完成堆栈或者队列的数据结构。内部是通过链表实现的，每个节点都含有前驱节点、后驱节点。

```java
/**
 * LinkedList 含有的元素数量
 */
transient int size = 0;
/**
 * LinkedList 的第一个节点
 */
transient Node<E> first;
/**
 * LinkedList 的最后一个节点
 */
transient Node<E> last;

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

### 添加元素

LinkedList 实现了 List 和 Qeque 两个接口，所以它的添加方法有多个

```java
/**
 * Deque 接口的添加元素方法，在LinkedList的结尾处添加，除此之外，还有 offerFirst offerLast
 */
public boolean offer(E e) {
    return add(e);
}
/**
 * List 接口的添加元素方法，在LinkedList的结尾处添加，除此之外，还有addFirst addLast
 */
public boolean add(E e) {
    linkLast(e);
    return true;
}
/**
 * Deque 接口的添加元素方法，在LinkedList的头部添加
 */
public void push(E e) {
    addFirst(e);
}
/**
 * 「尾插法」添加元素的方法
 */
void linkLast(E e) {
     final Node<E> l = last;
     // Node(Node<E> prev, E element, Node<E> next)
     final Node<E> newNode = new Node<>(l, e, null);
     last = newNode;
     if (l == null)
         first = newNode;
     else
         l.next = newNode;
     size++;
     modCount++;
 }
/**
 * 「头插法」添加元素的方法
 */
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

LinkedList 实现了 List 和 Qeque 两个接口，所以它的删除方法有多个

```
/**
 * remove 和 pop 都是从头部取值，如果为空，会抛出异常
 */
public E remove() {
    return removeFirst();
}
public E pop() {
    return removeFirst();
}
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
private E unlinkFirst(Node<E> f) {
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

## HashMap

Map 接口是所有存放键值对容器的根接口。其中，键可以是任意类型的对象，但不能有重复的键，每一个键对应一个值。具体实现 Map 接口的实现类主要有：HashMap 、LinkedHashMap、TreeMap：

- HashMap：内部通过数组加链表的形式来存放数据，具体存放位置由 hash 值来影响

- LinkedHashMap：内部将 HashSet 的数组换成了链表，由此支持了对其中的元素进行插入顺序的提取

- TreeMap：内部通过红黑树完成数据的有序，可以通过指定比较器来确定集合中的元素顺序

HashMap 内部是通过数组和链表结合的方式来实现保存数据的。具体来说，HashMap 内部维护了一个 Entry 数组，每个 Entry 对象包含了键值对的信息，以及指向下一个 Entry 对象的指针。它有三个常见构造函数

```java
//capacity 默认初始值 16，且 capacity 的值只能是 2 的次幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

//capacity 的最大值
static final int MAXIMUM_CAPACITY = 1 << 30;

//默认加载因子数值，当元素个数所占容量超过加载因子比例，就需要扩容
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//链表转为红黑树的分界值
static final int TREEIFY_THRESHOLD = 8;

//红黑树转为链表的分界值
static final int UNTREEIFY_THRESHOLD = 6;

//链表转红黑树时，所需的最小元素数量
static final int MIN_TREEIFY_CAPACITY = 64;

//存放所有元素的 Node 数组
transient Node<K,V>[] table;

//HashMap 存储的元素个数
transient int size;

//加载因子
final float loadFactor;

//下一次需要扩容的值 (capacity * loadFactor).
int threshold;

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

### 添加元素

HashMap 添加元素的整体流程是：

- 根据键的 hash 值计算出数组下标
- 如果该下标对应的 Entry 对象为空，就新建一个 Entry 对象并插入到数组
- 如果该下标对应的 Entry 对象已经有其他的键值对，就通过链表结构将新的键值对插入到链表的尾部
- 插入元素后，如果 size 到达扩容的临界值，进行扩容

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

//计算 key 的hash 值，为了减少碰撞率，让 hash 值的低16位也参与影响了最后的返回值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
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
                    // 找到链表结尾，并插入，binCount>8 了，那说明链表长度也超过了转为树的分界值8
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

resize() 的作用是是对 HashMap 中的数组进行扩容和重新分配，扩容的方式是将数组的长度翻倍，并将原有的元素重新分配到新数组中，重新计算它们的数组下标。调用 resize() 有四种情况：

- 第一次添加元素，此时 table=null&oldCap=0 
  - 空构造函数创建的HashMap oldThr=0 ，设置 newCap=DEFAULT_INITIAL_CAPACITY&newThr=(int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY)，返回新的 newCap 大小的新数组
  - 指定大小的构造函数创建的 HashMap oldThr=tableSizeFor()返回值，设置 newCap=oldThr&newThr=newCap*loadFactor，返回新的 newCap 大小的新数组

- 非第一次添加元素，此时 table !=null&oldCap=table.length，判断 oldCap 是否超过MAXIMUM_CAPACITY
  - 超过就不再扩容，设置 threshold=Integer.MAX_VALUE，返回原来的 table 数组
  - 没有超过，设置 newThr=oldThr*2&newCap=oldCap\*2，返回新的 newCap 大小的新数组

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

## LInkedHashMap

LinkedHashMap 是一个继承自 HashMap 的哈希表实现，它在哈希表的基础上维护了一个双向链表，用于保持元素的插入顺序或访问顺序。这使得迭代时可以按照顺序访问元素。

```java
static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
    //增加before和after节点
    LinkedHashMapEntry<K,V> before, after;
    LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

LinkedHashMap 支持两种顺序：插入顺序和访问顺序。

- 插入顺序是指按照元素被添加到哈希表中的顺序进行排序
- 访问顺序是指按照元素被访问的顺序进行排序。当以访问顺序构建 LinkedHashMap 时，每次访问一个元素，这个元素都会被移到链表的末尾

LinkedHashMap并没有重写 put/get 方法，但是其重写了 afterNodeAccess() 用于设置 after 和 before 节点

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMapEntry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

LinkedHashMap 可以方便地用于实现 LRU（Least Recently Used，最近最少使用）缓存策略。通过重写 removeEldestEntry 方法，当前长度> 缓存长度时返回true，即可在达到缓存容量限制时自动移除链表头部的最老元素。

## SparseArray

SparseArray 是 Android 框架提供的一个轻量级的数据结构，用于替代 HashMap<Integer, Object>。它用两个单独的数组来存储 key 和 values，并对键数组进行排序以实现二分查找。

```java
public class SparseArray<E> implements Cloneable {
    private static final Object DELETED = new Object();
    private boolean mGarbage = false;

    private int[] mKeys;
    private Object[] mValues;
    private int mSize;
    ......
}
```

### 删除

SparseArray 提供了 2 种删除方式，通过二分查找找到对应元素下标，删除只是将待删除元素标记为 DELETED，然后将 mGarbage 设置为 true，然后在 gc 过程中完成删除元素的过程。

```java
public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}

public void removeAt(int index) {
    if (index >= mSize && UtilConfig.sThrowExceptionForUpperArrayOutOfBounds) {
        throw new ArrayIndexOutOfBoundsException(index);
    }
    if (mValues[index] != DELETED) {
        mValues[index] = DELETED;
        mGarbage = true;
    }
}
```

### 添加

put() 方法主要做了以下操作：

- 通过二分查找在 mKeys 数组中查找对应的 key
- 查找成功则直接更新 key 对应的 value，而且不返回旧值
- 如果当前要插入的 key 索引上的值为 DELETE，直接覆盖
- 如果需要执行 gc 方法 & 需要扩大数组容量，则会执行 gc 方法
- 由于 gc 方法会改变元素的位置，因此会重新计算插入的位置并插入

```java
public void put(int key, E value) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
    if (i >= 0) {
        //正常找到
        mValues[i] = value;
    } else {
        i = ~i;
        if (i < mSize && mValues[i] == DELETED) {
           //标记为DELETE直接覆盖
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }
        //该位置有值，gc移除标记为DELETE元素，然后移动数组
        if (mGarbage && mSize >= mKeys.length) {
            gc();
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }
        //扩容容量为原容量的 2 倍
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```

### 回收

gc 将 mValue 数组中还未标记为 DELETED 的元素以及对应下标的 mKeys 数组中的元素移动到数组的前面，保证数组前面的数据都是有效数据，而不是存储着 DELETED：

```java
private void gc() {
    int n = mSize;
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;
    for (int i = 0; i < n; i++) {
        Object val = values[i];
        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                values[i] = null;
            }
            o++;
        }
    }
    mGarbage = false;
    mSize = o;
}
```

