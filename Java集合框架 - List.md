##### Q1：List 接口下有哪些常见方法？

「添加」：add(int index, E e) add(E e) set(int index, E e) 

「删除」：remove(int index) remove(E e) clear()

「获取」：isEmpty() size() get(int index) contains(Object o) indexOf(Object o) lastIndexOf(Object o)

  其他：sort(Comparator c) subList(int startIndex, int endIndex) toArray()

##### Q2：ArrayList 的常见成员变量及含义？

```java
/**
 * 实际上存放元素的容器，capacity 就是这个数组的长度。
 * 当构造函数中没有指定数组大小时，此数组会指向一个空数组，
 * 此后添加一个元素时，capacity 会增加到 DEFAULT_CAPACITY
 */
transient Object[] elementData; // non-private to simplify nested class access

/**
 * elementData 中实际存放的元素个数
 */
private int size;

/**
 * ArrayList 容量的默认值
 */
private static final int DEFAULT_CAPACITY = 10;

/**
 * 空数组，在没有添加元素之前，一般都是一个
 */
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
 * 一个共享的空数组对象，用于构造函数时没有指定大小。
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * elementData 中存放元素的上限
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

ArrayList 的构造函数

```java
/**
 * elementData 指向空数组
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

/**
 * 当指定的容量大小是 0，处理方式和空参构造函数保持一致
 * 大于 0 时，elementData 指向一个指定大小的数组
 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    }
}
```

ArrayList 的空参构造函数会给 elementData 赋值一个空数组，此时 size 仍然是 0 ，直到调用 add() 方法时 elementData 才会创建一个长度是10的数组。

ArrayList 指定初始容量的构造函数会对入参进行一次检查，小于 0 会抛出异常，等于 0  会指向一个空数组，只有大于 0 ，才会赋值一个指定大小的数组。

##### Q3：ArrayList 添加/删除元素源码分析？

ArrayList 添加元素有两个重载方法：add(E e)  和 add(int index, E element) 。

首先，熟悉一下 add() 方法的主要流程：

```java
/**
 * 「尾插法」添加新的元素
 */
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

先在这里整理一下流程，当调用 add(E e) 方法时，会先判断当前 elementData 数组的容量是否能够支持。如果无法支持，就会进行扩容：首先对旧容量扩大1.5倍，如果还是无法满足需求，就直接扩容至最小需求容量。无论是扩容还是直接赋值最小需求容量后的大小如果大于了 MAX_ARRAY_SIZE ，根据最小需求容量是否大于 MAX_ARRAY_SIZE 决定是 Integer.MAX_VALUE 还是 MAX_ARRAY_SIZE；如果扩容还是直接赋值最小需求容量后的大小如果小于 MAX_ARRAY_SIZE ，那就是最终的数组大小。

add(int index, E element)相较于add(E e) ，增加了入参检查和数组赋值操作System.arraycopy()。

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

ArrayList 删除元素有很多重载函数，了解下 remove(int index) 其他的都是大同小异。

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

remove() 的实现，底层还是通过数组移动复制，来覆盖 index 位置上的元素。有一点要注意点是在复制移动后，还手动的将结尾多出来的元素设置为 null，从而让 GC 顺利完成内存清理。

##### Q4：System.arrayCopy 和 Arrays.copyOf 两个方法的区别和联系？

```java
public static native void arrayCopy(Object src, int srcPos, Object dest, int destPos, int length);
```

System.arrayCopy() 是一个系统的原生方法，它的参数中需要一个目标数组，然后将原数组拷贝到目标数组中。可以选择拷贝的起点和长度。

```java
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

Arrays.copyOf() 内部调用了System.arraycopy()  是方法内部新建一个数组，并返回该数组。

##### Q5：ArrayList 为什么在遍历中直接删除会报异常？

在遍历中删除 ArrayList 中某个元素，都需要先获取 Iterator 对象，然后再调用 Iterator.remove 来删除，而不能在遍历中调用 List.remove 方法来删除元素。

```java
/**
 * List 接口获取迭代器
 */
public Iterator<E> iterator() {
    return new Itr();
}

private class Itr implements Iterator<E> {
    // ArrayList 中的元素个数
    protected int limit = ArrayList.this.size;
    // 下一个要返回的 index
    int cursor; 
    // 上一个返回元素的 index
    int lastRet = -1;
    // 期望的修改次数
    int expectedModCount = modCount;
    /**
	 * 如果下一个返回元素的 index 小于总的元素个数，说明还有下一个
	 */
    public boolean hasNext() {
        return cursor < limit;
    }
    /**
	 * 获取下一个元素
	 */
    public E next() {
        // 实际修改值和期望修改值不一样，会抛异常，也是上面问题的答案
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        int i = cursor;
        // index 位置判断
        if (i >= limit)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        // 当返回值后，cursor 就指向了下一个元素的 index
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }
    /**
	 * 删除元素
	 */
    public void remove() {
        // 上一个返回元素的 index<0 抛异常
        if (lastRet < 0)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        try {
            // 删除上一个返回的元素（涉及到数组的复制和移位）
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
    ...
}
```

##### Q6：LinkedList 常见成员变量及含义？

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

##### Q7：LinkedList 添加/删除元素源码分析？

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

```java
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