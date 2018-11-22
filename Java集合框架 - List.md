ArrayList 通过不同构造函数创建后，初始状态什么样的

ArrayList 添加和删除元素源码分析

ArrayList 的扩容机制，包括怎么扩容，扩容最大值

System.arraycopy 和 Arrays.copyOf 两个方法分析

ArrayList 的成员常量

```java
/**
 * Default initial capacity.
 */
private static final int DEFAULT_CAPACITY = 10;

/**
 * Shared empty array instance used for empty instances.
 * 
 */
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
 * Shared empty array instance used for default sized empty instances. We
 * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
 * first element is added.
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * The maximum size of array to allocate.
 * Some VMs reserve some header words in an array.
 * Attempts to allocate larger arrays may result in
 * OutOfMemoryError: Requested array size exceeds VM limit
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

ArrayList 的成员变量

```java
/**
 * The array buffer into which the elements of the ArrayList are stored.
 * The capacity of the ArrayList is the length of this array buffer. Any
 * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
 * will be expanded to DEFAULT_CAPACITY when the first element is added.
 */
transient Object[] elementData; // non-private to simplify nested class access

/**
 * The size of the ArrayList (the number of elements it contains).
 *
 * @serial
 */
private int size;
```

ArrayList 存放数据实际上就是存放在数组 elementData 中，size 是数组中存放数据的数量。

##### ArrayList 的构造函数

```java
/**
 * Constructs an empty list with an initial capacity of ten.
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

/**
 * Constructs an empty list with the specified initial capacity.
 *
 * @param  initialCapacity  the initial capacity of the list
 * @throws IllegalArgumentException if the specified initial capacity
 *         is negative
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

ArrayList 的空参构造函数会给 elementData 赋值一个空数组，此时 size 仍然是 0 ，直到调用 add() 方法时才会创建一个初始容量是 10的数组给 elementData。

ArrayList 指定初始容量的构造函数会对入参进行一次检查，小于 0 会抛出异常，等于 0  会指向一个空数组，只有大于 0 ，才会赋值一个指定大小的数组。

##### ArrayList 的添加元素流程

ArrayList 添加元素有两个重载方法：add(E e)  和 add(int index, E element) ，后者相较于前者，多了检查入参 index 的方法rangeCheckForAdd() 和数组复制的操作 System.arraycopy()。

首先，熟悉一下 add() 方法的主要流程：

```java
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return <tt>true</tt> (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 确认添加元素后内部数组容量是否正常
    elementData[size++] = e;
    return true;
}

/**
 * Inserts the specified element at the specified position in this
 * list. Shifts the element currently at that position (if any) and
 * any subsequent elements to the right (adds one to their indices).
 *
 * @param index index at which the specified element is to be inserted
 * @param element element to be inserted
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public void add(int index, E element) {
    rangeCheckForAdd(index);
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    elementData[index] = element;
    size++;
}

/**
 * A version of rangeCheck used by add and addAll.
 */
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

/**
 * 判断 elementData 数组的容量是否能支持当前容量+1
 */
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

/**
 * 计算 elementData 的需要的最小容量
 */
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

/**
 * 确认是否需要扩展数组
 */
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

先在这里整理一下流程，当调用 add() 方法时，会先判断当前 elementData 数组的容量是否能够支持。

注意：当我们通过上文提到的两个构造函数创建 ArrayList 对象时，size 的值一直是 0，所以在 calculateCapacity() 方法时会有两种情况，对于空构造函数创建的 ArrayList 对象，会返回 Math.max(DEFAULT_CAPACITY, minCapacity) 也就是 10，而指定了容量大小的则会返回 size +1 也就是1。

然后就来到 ensureExplicitCapacity() 确认是否要扩展数组了。指定了容量大小的 ArrayList 对象，需求容量是1，而实际空数组的长度比1 大。暂时就不会扩容。而空构造函数创建的 ArrayList 对象。需求容量是10，而实际空数组的长度是0，此时就需要通过 grow() 方法对 elementData 数组进行扩容。

```java
/**
 * Increases the capacity to ensure that it can hold at least the
 * number of elements specified by the minimum capacity argument.
 *
 * @param minCapacity the desired minimum capacity
 */
private void grow(int minCapacity) {
    // oldCapacity = 0 ,minCapacity = 10
    int oldCapacity = elementData.length;
    // newCapacity = 0 ,minCapacity = 10
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        // newCapacity = 10 ,minCapacity = 10
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

/**
 * 对于 newCapacity 超过了 Java 要求的最大数组长度，就对比下 minCapacity
 */
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

上面注释的内容是前面空构造函数创建的 ArrayList 对象。需求容量是10，而实际空数组的长度是 0 的场景。

整体整理下来 grow() 方法就是把 elementData 数组的长度增加到 1.5 倍，再和需求容量(minCapacity)比较，如果仍然小，newCapacity 就取 minCapacity 的值。然后再拿 newCapacity 的大小和 Java 要求的数组最大容量对比，如果超出了，就根据需求容量(minCapacity)取 Integer.MAX_VALUE 还是 MAX_ARRAY_SIZE。然后通过 Arrays.copyOf() 扩展 elementData 数组。

到了这里，elementData 数组的长度问题就已经解决了，最后就是把元素真正的添加到数组中，对于 add(E e) ，把元素放到 size+1的位置就可以，但对于 add(int index, E element) ，我们需要把 index 及以后的元素往后移动一位，这时候需要调用 System.arraycopy()。

那么  Arrays.copyOf () 和 System.arraycopy() 两个方法有什么区别和联系呢？

```java
/**
 *  Arrays.copyOf
 */
public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}

public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0, Math.min(original.length, newLength));
    return copy;
}

/**
 *   System.arraycopy() 是一个原生方法
 */
public static native void arraycopy(Object src,  int  srcPos, Object dest, int destPos, int length);
```

System.arraycopy() 需要目标数组，将原数组拷贝到你自己定义的数组里，而且可以选择拷贝的起点和长度以及放入新数组中的位置；Arrays.copyOf() 内部调用了System.arraycopy()  是方法内部新建一个数组，并返回该数组。

##### ArrayList 的删除元素流程

ArrayList 删除元素有很多重载函数，了解下 remove(int index) 其他的都是大同小异。

```java
 /**
  * Removes the element at the specified position in this list.
  * Shifts any subsequent elements to the left (subtracts one from their
  * indices).
  *
  * @param index the index of the element to be removed
  * @return the element that was removed from the list
  * @throws IndexOutOfBoundsException {@inheritDoc}
  */
 public E remove(int index) {
     rangeCheck(index);
     modCount++;
     E oldValue = elementData(index);
     int numMoved = size - index - 1;
     if (numMoved > 0)
         System.arraycopy(elementData, index+1, elementData, index,numMoved);
     elementData[--size] = null; 
     return oldValue;
 }
```

remove() 的实现，底层还是通过数组移动复制，来覆盖 index 位置上的元素。有一点要注意点是在复制移动后，还手动的将结尾多出来的元素设置为 null，从而让 GC 顺利完成内存清理。

ArrayList 是如何通过 iterator() 来完成删除操作的

```java
public Iterator<E> iterator() {
    return new Itr();
}

/**
 * An optimized version of AbstractList.Itr
 */
private class Itr implements Iterator<E> {
    int cursor;       // 指向下一个遍历的元素
    int lastRet = -1; // 指向最新返回的元素
    int expectedModCount = modCount;
    
    Itr() {}
    
    public boolean hasNext() {
        return cursor != size;
    }

    public E next() {
        // 如果在 iterator 遍历的过程中，通过 ArrayList remove 方法删除元素，expectedModCount ！= modCount 就会报错
        checkForComodification();
        // 保存当前元素 index
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementDa
        if (i >= elementData.length)
            throw new ConcurrentModificationException()
        // cursor 指向下一个元素 index
        cursor = i + 1;
        // lastRet 赋值为最新返回的元素 index
        return (E) elementData[lastRet = i];
    }
    
    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();
        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            // remove 方法删除元素，会 同步 expectedModCount 和 modCount
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException()
        }
    }
}
```

LInkedList  内部是怎么存放数据的

LInkedList  实现了 List 和 Queue 两个接口，添加和删除操作怎么实现

```java
/**
 * Adds the specified element as the tail (last element) of this list.
 *
 * @param e the element to add
 * @return {@code true} (as specified by {@link Queue#offer})
 * @since 1.5
 */
public boolean offer(E e) {
    return add(e);
}

/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #addLast}.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    linkLast(e);
    return true;
}

 /**
  * Links e as last element.
  */
 void linkLast(E e) {
     final Node<E> l = last;
     final Node<E> newNode = new Node<>(l, e, null);
     last = newNode;
     if (l == null)
         first = newNode;
     else
         l.next = newNode;
     size++;
     modCount++;
 }
```

LinkedList 的 add 和 offer 方法是完全等效的，都是在链表的尾部添加元素，具体的实现是 linkLast() 方法来完成，LinkedList 中每个节点是这样的

```java
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

其中 next 指向下一个节点， prev 指向前一个节点。