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


