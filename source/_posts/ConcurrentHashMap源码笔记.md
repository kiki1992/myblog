---
title: ConcurrentHashMap源码笔记
date: 2017-12-02 17:56:32
summary: 本篇对ConcurrentHashMap接口源码做了简单分析及整理
tags: Java Collections Framework
---

# ConcurrentHashMap源码笔记
## 主要特性
> * 1.ConcurrentHashMap支持完全并发的检索操作以及较高并发的更新操作。本类遵循同HashTable类一样的机能规格，针对HashTable的各个方法都有对应版本的实现。另外，尽管本类的所有操作都是线程安全的，查询操作并不会导致锁定，也不支持加锁整张表来阻止任何访问。当仅关心线程安全而不是具体的同步机制时本类可以完全兼容HashTable。
> * 2.检索操作(包括get)通常不会阻塞，因此可能会和更新操作(包括put和remove)部分重叠。检索操作会反映与其相关的最近一次更新操作结果。(换句话说，一次对指定key的更新操作happens-before任何(非null)通过同样key获取到更新后value的检索操作)。
> * 3.诸如putAll和remove这样的聚合操作，并发的检索操作可能只能反映部分新增和移除结果。类似的，Iterators, Spliterators 以及 Enumerations 返回的元素仅反映iterator/enumeration创建时或创建后哈希表某一时刻的状态。
> * 4.需要注意的是，诸如size, isEmpty这样的聚合状态方法以及containsValue方法返回的都是某个时刻的状态，因此只能用来作为检测或评估之用，而不应该作为程序中的控制条件使用。

## 构造器
ConcurrentHashMap提供了无参，可以指定初始容量/负载因子/并发级别(并发更新线程数)，以及以Map实例作为入参的构造器。每个构造器的具体特性参考下面的注释。
**需要注意的是，作为入参传入的负载因子只会影响实际初始容量的大小**
```java
// 使用默认的初始容量(16)来创建一个空map
public ConcurrentHashMap() {
}

// 根据指定初始容量来创建一个空map
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    // 如果指定初始容量>=最大容量的1/2就以最大容量作为初始容量
    // 否则就已>=并且最接近(指定初始容量 + 指定初始容量/2 + 1)的2的幂值作为初始容量
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    // sizeCtl变量作为触发下次扩容的临界值
    this.sizeCtl = cap;
}

// 创建一个与指定map实例拥有相同mapping的对象
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

// 根据指定初始容量，负载因子，以及默认的并发级别(1)创建一个空map
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

// 根据指定初始容量，负载因子，以及并发级别创建一个空map
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // 保证初始容量至少比并发更新线程数大
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

## 内部类
Node -- 键/值映射
> 1.本类不支持对value的修改，适用于只读遍历。
> 2.哈希值为负数的子类有特殊用途，如TreeNode,ForwardingNode等。

```java
static class Node<K,V> implements Map.Entry<K,V>
```
#### 内部变量
```java
final int hash; // 哈希值
final K key; // 键
volatile V val; // 值 volatile保证值总是最新的
volatile Node<K,V> next; // 下一节点 volatile保证next的变化总是能立即反映
```
#### 重要方法/构造器
```java
// 不支持value修改
public final V setValue(V value) {
    throw new UnsupportedOperationException();
}

// find方法默认实现，在子类中重写
Node<K,V> find(int h, Object k) {
    Node<K,V> e = this;
    if (k != null) {
        do {
            K ek;
            if (e.hash == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
        } while ((e = e.next) != null);
    }
    return null;
}
```
***
ForwardingNode -- 临时转发节点
> 1.本类是Node类的一个特殊子类，哈希值固定-1，仅存在于扩容过程中。
> 2.插入该节点代表原table的节点已经移动至新的table，新table通过.nextTable获取。
> 3.遍历操作过程中发现此类型节点会使遍历操作转发至新table，put操作过程中发现此类型节点则可以协助扩容。

#### 内部变量
```java
final Node<K,V>[] nextTable; // 指向新table -- final修饰保证指向一经确定不会改变
```
#### 重要方法/构造器
```java
// 除了哈希值固定-1以外，其它实例域都为null
ForwardingNode(Node<K,V>[] tab) {
    super(MOVED, null, null, null);
    this.nextTable = tab;
}

// 重写父类的find方法，会基于新table执行查找操作
Node<K,V> find(int h, Object k) {
    // loop to avoid arbitrarily deep recursion on forwarding nodes
    outer: for (Node<K,V>[] tab = nextTable;;) {
        Node<K,V> e; int n;
        // 先检查table或者table指定下标是否为空
        if (k == null || tab == null || (n = tab.length) == 0 ||
            (e = tabAt(tab, (n - 1) & h)) == null)
            return null;
        for (;;) {
            int eh; K ek;
            // 当前节点符合条件则直接返回
            if ((eh = e.hash) == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
            // 哈希值<0代表特殊节点，需要根据节点类型区分操作
            if (eh < 0) {
            	// 再次遇到临时转发节点，将查找操作转发至新table，否则调用节点类型对应find方法。
                if (e instanceof ForwardingNode) {
                    tab = ((ForwardingNode<K,V>)e).nextTable;
                    continue outer;
                }
                else
                    return e.find(h, k);
            }
            if ((e = e.next) == null)
                return null;
        }
    }
}
```
***
Traverser
> 1.本类为ConcurrnetHashMap的一些方法提供遍历支持，例如ContainsValue，同时也是其它iterator和spliterator的基类。
> 2.通常，遍历操作是挨个桶执行的。但是如果发生了扩容操作，那就需要遍历新table中的两个桶，分别是下标为当前index的桶和下标为index+baseSize的桶。这是因为table扩容后会重新计算原桶中元素的下标，这会使部分元素被分配到index+baseSize的桶中。
```java
static class Traverser<K,V>
```
#### 内部变量
```java
Node<K,V>[] tab;        // 当前遍历的table，扩容时会更新
Node<K,V> next;         // 下一节点
TableStack<K,V> stack, spare; // 遇到ForwardingNode时保存/恢复原table
int index;              // 下一个桶的位置
int baseIndex;          // table初始index
int baseLimit;          // table初始index上限
final int baseSize;     // table初始大小
```
#### 重要方法/构造器

```java
// 尝试获取下一个节点元素，如果遇到ForwardingNode会保存当前遍历的table信息，并在完成新table指定位置遍历后恢复之前保存的table信息。 
final Node<K,V> advance() {
    Node<K,V> e;
    // 尝试获取下一节点
    if ((e = next) != null)
        e = e.next;
    for (;;) {
        Node<K,V>[] t; int i, n;  // must use locals in checks
        // 获取到非null节点则直接返回
        if (e != null)
            return next = e;
        // 边界检验--如果当前遍历的table为空或者已经完成遍历则直接返回null
        if (baseIndex >= baseLimit || (t = tab) == null ||
            (n = t.length) <= (i = index) || i < 0)
            return next = null;
        // 尝试获取接下来需要遍历的桶
        // 如果桶非空并且桶内存放的是特殊节点则进行特殊处理    
        if ((e = tabAt(t, i)) != null && e.hash < 0) {
        	// 如果桶内存放了ForwardingNode，则保存当前遍历table状态并将遍历操作转发到新table
            if (e instanceof ForwardingNode) {
                tab = ((ForwardingNode<K,V>)e).nextTable;
                e = null;
                pushState(t, i, n);
                continue;
            }
        	// 如果桶内存放了TreeBin，则获取红黑树链表形式的头节点进行遍历
            else if (e instanceof TreeBin)
                e = ((TreeBin<K,V>)e).first;
        	// 其它节点(ReservationNode)，没有实际数据
            else
                e = null;
        }
        // table遍历状态保存栈不为空，则尝试出栈操作，前提是已经遍历玩当前table的两个桶
        if (stack != null)
            recoverState(n);
        // 标记下一个需要遍历的桶下标，在当前桶遍历完成之后才会用到新的index
        else if ((index = i + baseSize) >= n)
            index = ++baseIndex; // visit upper slots if present
    }
}

// 当获取到ForwardingNode时保存当前遍历table状态
private void pushState(Node<K,V>[] t, int i, int n) {
	 // 如果spare不为null，则复用为栈顶
    TableStack<K,V> s = spare;  // reuse if possible
    if (s != null)
        spare = s.next;
    else
        s = new TableStack<K,V>();
    // 保存当前table的遍历状态会执行入栈操作
    s.tab = t;
    s.length = n;
    s.index = i;
    s.next = stack;
    stack = s;
}

// 尝试恢复上一次保存的table遍历状态，前提是已经完成当前table两个桶的遍历
private void recoverState(int n) {
    TableStack<K,V> s; int len;
    // index += (len = s.length)) >= n 表示当前table没有下一个需要遍历的桶了
    while ((s = stack) != null && (index += (len = s.length)) >= n) {
    // 恢复栈顶保存的table遍历状态，并将出栈元素保存在spare中以便复用
        n = len;
        index = s.index;
        tab = s.tab;
        s.tab = null;
        TableStack<K,V> next = s.next;
        s.next = spare; // save for reuse
        stack = next;
        spare = s;
    }
    // 标记下一个需要遍历的桶下标，在当前桶遍历完成之后才会用到新的index
    if (s == null && (index += baseSize) >= n)
        index = ++baseIndex;
}

```

## 主要方法
### 检索方法
#### get
由于ConcurrentHashMap不支持null作为key/value，返回null即表示不存在指定key对应的value，这一点和HashMap不同。下面先贴一下整体代码，再逐步分析具体实现。
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
下面对get方法源码做一下简单的分析:

首先看内部方法spread，主要是一些位操作，目的是使数据更分散，减少碰撞。
因为在插入元素时会调用spread方法对key的哈希值进行处理，因此检索元素时也需要调用本方法来保证一致性。
* 1.将h的高16位和低16位作异或操作，兼顾高位和低位数据来增加结果的随机性以减少碰撞发生。如果没有这一步操作，在哈希表长度较小时，通过哈希值和数组长度计算下标的操作将不会利用高位数据，这样就可能会增加数据碰撞的几率。
* -- h ^ (h >>> 16)
* 2.将操作1的结果和0x7fffffff做与操作置最高位为0，保证结果一定是整数。这是因为负数值有特殊的用途(比如ForwardingNode的哈希值固定为-1)
* -- (h ^ (h >>> 16)) & HASH_BITS
```java
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
***
接着是针对哈希表的一些判断，首先检查哈希表是否不为空，接着看key哈希值(处理后)对应的数组下标是否存在mapping元素。如果两者都不满足，就直接返回null。
```java
if ((tab = table) != null && (n = tab.length) > 0 &&
   (e = tabAt(tab, (n - 1) & h)) != null) 
```
***
最后就是在对应下标查找元素的过程了。代码如下:
首先会检查当前下标对应的首节点是否就是要找的节点，是就直接返回首节点。
接着判断首节点hash值，hash值<0说明是特殊节点(TreeNode,ForwardingNode..)，需要调用对应的find方法查找节点，hash值>=0则说明是链表节点，直接遍历查找。
```java
if ((eh = e.hash) == h) {
  if ((ek = e.key) == key || (ek != null && key.equals(ek)))
      return e.val;
}
// 特殊节点
else if (eh < 0)
  return (p = e.find(h, key)) != null ? p.val : null;
// 链表节点  
while ((e = e.next) != null) {
  if (e.hash == h &&
      ((ek = e.key) == key || (ek != null && key.equals(ek))))
      return e.val;
}
```
#### containsKey
该方法直接调用get方法并判断其返回值。
```java
public boolean containsKey(Object key) {
    return get(key) != null;
}
```
#### containsValue
该方法需要遍历整个map，效率会比containsKey慢很多(根据key查找可以根据哈希值直接命中对应的哈希表下标，查找范围会小很多)。
这里重点关注内部类Traverser
```java
public boolean containsValue(Object value) {
    if (value == null)
        throw new NullPointerException();
    Node<K,V>[] t;
    if ((t = table) != null) {
        Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
        for (Node<K,V> p; (p = it.advance()) != null; ) {
            V v;
            if ((v = p.val) == value || (v != null && value.equals(v)))
                return true;
        }
    }
    return false;
}

```



