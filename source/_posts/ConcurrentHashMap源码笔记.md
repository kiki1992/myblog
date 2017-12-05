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
Traverser
```java
static class Traverser<K,V>
```
###内部变量
```java
Node<K,V>[] tab;        // current table; updated if resized
Node<K,V> next;         // the next entry to use
TableStack<K,V> stack, spare; // to save/restore on ForwardingNodes
int index;              // index of bin to use next
int baseIndex;          // current index of initial table
int baseLimit;          // index bound for initial table
final int baseSize;     // initial table size
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
首先会检查当前下标对应的首节点是否就是要找的节点，是就直接返回首节点，否则遍历当前下标下的Node链表/红黑树查找指定key对应的mapping节点。
```java
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



