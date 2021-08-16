- [Java数据结构](#java数据结构)
  - [1.	ArrayList和LinkedList的区别](#1arraylist和linkedlist的区别)
  - [2.	ArrayList的扩容机制](#2arraylist的扩容机制)
  - [3.	List, Set, Map的区别](#3list-set-map的区别)
  - [4.	HashMap的底层原理](#4hashmap的底层原理)
  - [5.	LinkedHashMap的底层原理](#5linkedhashmap的底层原理)
  - [6.	ConcurrentHashMap的数据结构](#6concurrenthashmap的数据结构)
  - [7.	ConcurrentHashMap和Hashtable的区别](#7concurrenthashmap和hashtable的区别)
  - [8.	谈谈并发容器](#8谈谈并发容器)
  - [9.	HashMap的内存泄漏问题？](#9hashmap的内存泄漏问题)
  - [10. CopyOnWriteArrayList插入过程中再次插入会如何？](#10-copyonwritearraylist插入过程中再次插入会如何)

# Java数据结构
这一部分建议联系源代码观看，可以通过IDEA等IDE查看源码，才能有更深入的理解。

## 1.	ArrayList和LinkedList的区别
+ ArrayList的底层数据结构是Object[]，而LinkedList的底层数据结构是双向链表。
+ 因为ArrayList的底层数据结构是数组，因此，当插入一定量的数据时，ArrayList需要进行扩容，这需要为其分配一块新的连续空间，并且涉及到ArrayList的[扩容机制](#2-ArrayList的扩容机制)。
+ 此外，因为ArrayList的底层数据结构是数组，因此ArrayList实现了RandomAccess接口，因为数组支持随机访问。数组存储空间是连续的，因此可以根据数组地址以及数组下标直接计算出相应元素的地址。

## 2.	ArrayList的扩容机制
+ ArrayList默认的初始容量为10，而这个初始容量可以在构造方法当中自定义。
```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable {
    ...
    private static final int DEFAULT_CAPACITY = 10;
    ...
    public ArrayList(int initialCapacity) {
        ...
    }
    ...
}
```
+ 而当minCapacity大于当前容量时，就会进行扩容，每次扩容1.5倍，若仍然不足则扩容为minCapacity。
+ 当newCapacity大于MAX_ARRAY_SIZE之后，会调用hugeCapacity()方法：
	- 如果minCapacity > MAX_ARRAY_SIZE，则将容量设为Integer.MAX_VALUE。
	- 否则，将容量设为MAX_ARRAY_SIZE
+ 每次进行扩容，都需要通过Arrays.copyOf()方法将旧的数组深拷贝到新的数组当中，这样有较大的开销，因此，在插入大量数据之前，我们可以先调用ensureCapacity()方法来手动进行一次扩容。

具体扩容机制相关代码可见下方：（函数的顺序可能与源码不同）<br>
```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable {
    ...
    // 默认长度为10
    private static final int DEFAULT_CAPACITY = 10;
    private static final int MAX_ARRAY_SIZE = 2147483639;
    
    public ArrayList(int initialCapacity) {
        ...
    }
    
    ...

    public boolean add(E e) {
        ++this.modCount;
        // 调用private的add()方法
        this.add(e, this.elementData, this.size);
        return true;
    }

    private void add(E e, Object[] elementData, int s) {
        // 如果数组已满，进行扩容
        if (s == elementData.length) {
            elementData = this.grow();
        }

        elementData[s] = e;
        this.size = s + 1;
    }

    private Object[] grow(int minCapacity) {
        return this.elementData = Arrays.copyOf(this.elementData, this.newCapacity(minCapacity));
    }

    private int newCapacity(int minCapacity) {
        int oldCapacity = this.elementData.length;
        // 新容量为老容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 若新容量仍不足 minCapacity
        if (newCapacity - minCapacity <= 0) {
            if (this.elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
                return Math.max(10, minCapacity);
            } else if (minCapacity < 0) {
                throw new OutOfMemoryError();
            } else {
                // 则设置为 minCapacity
                return minCapacity;
            }
        } else {
            // 判断新容量是否超过 MAX_ARRAY_SIZE
            return newCapacity - 2147483639 <= 0 ? newCapacity : hugeCapacity(minCapacity);
        }
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) {
            throw new OutOfMemoryError();
        } else {
            // 如果minCapacity > MAX_ARRAY_SIZE，则将容量设为Integer.MAX_VALUE。
            // 否则，将容量设为MAX_ARRAY_SIZE
            return minCapacity > 2147483639 ? 2147483647 : 2147483639;
        }
    }

    ...
}
```

## 3.	List, Set, Map的区别
+ List存储的元素是有序、可重复的
+ Set存储的元素是无序、不可重复的
+ Map则是通过键值对进行存储的

## 4.	HashMap的底层原理
JDK1.8之后，Java的HashMap采用数组+链表+红黑树来实现。当插入一个数据，首先会计算其哈希值，并且根据哈希值计算出对应的数组下标。如果Map中已经存在这个key，就会替换value；而如果不存在这个key，就会插入新的节点。此时，如果出现哈希冲突，就会向链表当中继续插入值，而如果没有出现哈希冲突，就会在数组的相应位置放入一个节点。<br><br>
如果加入的元素数量达到阈值（负载因子*容量，默认负载因子为0.75），则会对数组进行扩容；而如果链表的长度大于阈值（默认为8），也会对哈希表进行扩容，这里分两种情况：
+ 如果数组的长度小于64，对数组进行扩容，将数组的长度扩容为2倍。数组的长度需要保持是2的倍数，因为只有当数组长度为2的倍数时，满足下式：<br>
$hash$ $\%$ $n$ $=$ $hash$ $\And$ $(n-1)$
<br>
从而可以用位运算来简化取余的计算，提高效率。<br><br>
因为数组是扩容为两倍，因此扩容之后Node的位置要么在原来的索引，要么位置在原索引+原数组大小处。因此，HashMap维护了两个临时链表引用，在分别用尾插法插入完成之后，再将这两个链表赋值到Node[]的对应位置。
+ 如果数组的长度大于64，则将链表转化为红黑树。这样可以将哈希冲突时查询的时间复杂度从O(n)降低为O(logn)，而如果后续通过删除，树中的元素数量小于6，则会将红黑树转换回链表。

相关代码如下：（为了便于理解，对部分顺序进行了调整）
```java
public class HashMap<K, V> extends AbstractMap<K, V> implements Map<K, V>, Cloneable, Serializable {
    // 默认初始容量为16
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    // 默认负载因子为0.75
    static final float DEFAULT_LOAD_FACTOR = 0.75F;
    // 链表长度超过8后，如果容量超过64，则将链表转化为红黑树
    static final int TREEIFY_THRESHOLD = 8;
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 红黑树中元素少于6时，退化回链表
    static final int UNTREEIFY_THRESHOLD = 6;

    // 哈希函数
    static final int hash(Object key) {
        int h;
        return key == null ? 0 : (h = key.hashCode()) ^ h >>> 16;
    }

    public V put(K key, V value) {
        return this.putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        HashMap.Node[] tab;   // 底层的Node[]
        int n;

        // 若数组未初始化，对其进行初始化
        if ((tab = this.table) == null || (n = tab.length) == 0) {
            n = (tab = this.resize()).length;
        }

        Object p;
        int i;
        // 若数组当前位置为空，新建Node节点
        if ((p = tab[i = n - 1 & hash]) == null) {
            tab[i] = this.newNode(hash, key, value, (HashMap.Node)null);
        } else {
            // 哈希冲突
            // 判断是否存在key相同的节点，以及树化的问题
            ...

            // 若已存在相同key的节点
            if (e != null) {
                V oldValue = ((HashMap.Node)e).value;
                // 根据onlyIfAbsent的设置决定是否替换
                if (!onlyIfAbsent || oldValue == null) {
                    ((HashMap.Node)e).value = value;
                }

                this.afterNodeAccess((HashMap.Node)e);
                return oldValue;
            }
        }

        ++this.modCount;
        if (++this.size > this.threshold) {
            this.resize();
        }

        this.afterNodeInsertion(evict);
        return null;
    }
}
```

## 5.	LinkedHashMap的底层原理
LinkedHashMap是HashMap的子类，因此，其主要的底层数据结构与HashMap相似，同样包含一个节点数组，链表、红黑树，不同的是，LinkedHashMap的节点类在HashMap的Node类基础上新增了指向上一个节点和下一个节点的指针，也就是说，LinkedHashMap在HashMap的基础上增加了一条双向链表。<br><br>
而LinkedHashMap的put操作时就增加了一步，即将节点接在链表的末尾。

## 6.	ConcurrentHashMap的数据结构
**JDK1.7之前**，ConcurrentHashMap是通过分段锁来实现线程安全的，它主要由Segment[], HashEntry[]以及链表组成。Segment继承了ReentrantLock，也就是说，每一个段都是一个可重入锁，当在某一段进行写入时，就会对这一段加锁。<br><br>
如果扩容数组，会将数组大小扩容为2倍，之后会重新计算每个HashEntry的位置。扩容时会找到一个在这个节点之后的节点新位置全部一致的节点，并将这个节点直接移动到新数组的相应位置；剩余的节点使用头插法逐渐插入。<br><br>
**JDK1.8之后**，ConcurrentHashMap不再使用Segment的分段锁机制，而是直接使用Node[]和链表、红黑树来实现，这种底层数据结构和HashMap非常相似。而与HashMap的区别在于，在操作的过程中加入了synchronized和CAS操作，从而保证了线程安全。在put的过程中，如果计算出的位置已经存在节点，会用**synchronized**锁住链表的首个节点，而不会影响Node数组中的其他Node，这样可以大幅度提高并发性。而如果计算出的位置暂时没有节点，会用**CAS**操作来插入新的节点。

## 7.	ConcurrentHashMap和Hashtable的区别
+ **ConcurrentHashMap**在JDK1.7及以前使用分段锁的机制，在JDK1.8之后则是通过synchronized锁住Node来实现线程安全，因此，无论是哪个版本的ConcurrentHashMap，在插入值时都只会给其中的一部分上锁。
+ 而**HashTable**每个对象使用synchronized修饰方法，这样调用同步方法时整个对象都会上锁，因此并发性能较差，目前已经基本不会使用Hashtable了。

## 8.	谈谈并发容器
我们知道，Java当中普通的容器类，比如HashMap, ArrayList, LinkedList等都是线程不安全的。那么对于这个问题，首先Collections提供了一系列的静态方法，可以这些collection变为线程同步的。这种方法虽然解决了线程不安全的问题，但是这种同步化的容器性能非常的低。因此，在多线程环境下，我们应该使用JUC包（java.util.concurrent）提供的并发容器。<br><br>
主要常用的并发容器有ConcurrentHashMap, CopyOnWriteArrayList, BlockingQueue等，其中：
+ **ConcurrentHashMap**是支持多线程并发操作的HashMap，它的底层数据结构是Node[]、链表、红黑树实现的。当调用put()方法时，如果计算出的位置之前没有节点，会通过CAS操作来新建一个节点放入数组；而如果计算出的位置已经存在一个链表，就会用synchronized锁住这个位置链表的首个节点。
+ **CopyOnWriteArrayList**的add, set等操作，都是**加锁**并通过**复制**原来的数组，在新的数组上修改之后，再将数据引用转为指向修改完成的数组。这里加ReentrantLock的目的是防止多个线程同时写入时拷贝出了多个副本，从而导致写入的数据丢失。而因为修改并不进行在原来的数组上，所以读取时并不需要进行同步和加锁，同时写线程也不会阻塞读线程。因此，CopyOnWriteArrayList很适合**读多写少**的多线程环境。
+ BlockingQueue则是阻塞队列，我们使用的线程池ThreadPoolExecutor就运用了阻塞队列。BlockingQueue是一个接口，常见的实现类有ArrayBlockingQueue和LinkedBlockingQueue，前者是通过数组实现的，而后者是通过链表实现的。

## 9.	HashMap的内存泄漏问题？
如果一个类的hashcode()方法被重写过，且hashcode()的返回值与成员变量有关，这个对象被设为key之后，再对其成员变量进行修改，就会导致内存泄漏。<br><br>
这是因为在put时，我们根据当时对象的成员变量计算了哈希值，并计算出它的位置，之后，key的内容被修改，如果再次插入或是调用get时，计算得到的哈希值就会不同，从而就无法正确找到原来插入的键值对。

## 10. CopyOnWriteArrayList插入过程中再次插入会如何？
CopyOnWriteArrayList的add方法中运用了ReentrantLock，因此在其中一个线程进行add操作的时候，其他线程会阻塞等待获取锁，之后才进行add操作。而对于同一线程，add操作是同步的，因此也需要等待前一个add操作结束之后才可以继续操作。<br><br>
CopyOnWriteArrayList的数组标记了volatile，可以防止指令重排，从而避免指令重排导致的，数组复制操作未完成前提前返回地址给引用的问题。

