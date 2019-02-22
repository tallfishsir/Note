# Java集合框架 - 概述

java的集合框架提供了数据的容器，主要包括了两种类型容器：一种是集合（Collection），存放的是元素；另一种是图（Map），存放的是键值对。

## Collection

Collection 接口是所有存放元素的容器的根接口，其中定义了对元素的基本操作，从增删改查的角度来整理的话：

- 增：add()  addAll()
- 删：remove() removeAll() clear()
- 查：size() isEmpty() contains() equals()

除此之外，还有转换为数组的 toArray() ，获取遍历器的  iterator()。

Collection 最常见的三个子接口是：List、Queue、Set

### List

List 接口定义的是一个允许重复的有序集合，相对于 Collection 接口，扩展了面向位置的操作。从增删改查的角度来整理的话，扩展了：

- 增：add(E e, int index)
- 删：remove(int index)
- 改：set(int index) sort()
- 查：get(int index) indexOf(E e) lastIndexOf(E e) subList(int, int)

具体实现 List 接口的实现类有：ArrayList、LinkedList、Vector。

ArrayList 内部是通过一个可以动态扩展大小的数组来存放元素的。

LinkedList 内部是通过链表来存放元素的，链表中的节点对象 Node 包含了数据值、前驱节点、后驱节点。

Vector 和 ArrayList 一样都是通过数组来存放元素，不同的是 Vector 的方法是线程安全的，实现方式是给所有方法添加 synchronized 关键字修饰。

鉴于 ArrayList 和 LinkedList 内部实现方式的不同，一般经常随机访问元素的情况选用 ArrayList，而经常插入删除元素的情况选用 LinkedList。

### Queue

Queue 接口定义的是一种先进先出的集合，相对于 Collction 接口，扩展了对集合头部、尾部操作，从增删改查的角度整理的话：

- 增：offer()
- 移除头部元素：remove() poll()
- 获取头部元素：element() peek()

如果集合为空，remove 会抛出异常，而 poll 会返回 null；element 会抛出异常，而 peek 会返回null。

当队列的长度是限制，使用 add 超出长度会抛出异常，offer 不会。

### Deque

Deque 接口是扩展 Queue 的双端队列，它支持在集合的两端插入和删除元素。前面介绍的 LinkedList 除了实现了 List 接口，还实现了 Deque 接口。从增删改查的角度整理：

- 增：addFirst() addLast() offerFirst() offerLast() push()
- 删：removeFIrst() removeLast() pollFIrst() pollLast()
- 查：getFirst() getLast() peekFirst() peekLast()

### Set

Set 接口定义的是一个不允许重复的无序集合，具体实现 Set 接口的实现类有：HashSet、LinkedHashSet、TreeSet。

HashSet 内部是通过 HashMap 实现，键值对中的值是一个 new Object() 对象。

LInkedHashSet 内部是通过 LinkedHashMap 实现，键值对中的值是一个 new Object() 对象。

TreeSet 内部是通过 TreeMap 实现，键值对中的值是一个 new Object() 对象。

## Map

Map 接口是所有存放键值对容器的根接口。其中，键可以是任意类型的对象，但不能有重复的键，每一个键对应一个值。具体实现 Map 接口的实现类主要有：HashMap 、LinkedHashMap、TreeMap。实际上，Set 接口的实现类在内部都是通过相应的 Map 实现类来完成功能的。

HashMap 内部是通过数组加链表的形式来存放数据，具体存放位置由 hash 值来影响。

LinkedHashMap 内部是将 HashSet 的数组换成了链表，由此支持了对其中的元素进行插入顺序的提取。

TreeMap 是有一个有序的结合，底层是红黑树，可以通过指定比较器来确定集合中的元素顺序。