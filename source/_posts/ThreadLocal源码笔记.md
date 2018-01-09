---
title: ThreadLocal源码笔记
date: 2018-01-08 15:28:32
summary: 本篇对ThreadLocal源码做了简单分析及整理。
tags: 多线程
---

## ThreadLocal源码笔记

```java
public class ThreadLocal<T> 
```

### 概要

> * ThreadLocal类用于提供线程本地变量存储的功能。所有访问ThreadLocal实例的线程都能得到独立与其他线程初始化的变量副本。
> * ThreadLocal类通常被定义为类的private static域来保存那些希望和线程联系在一起的状态，比如用户ID，事务ID之类的东西。
> * 每个线程都会持有线程本地变量副本的隐式引用，只要满足线程存活和ThreadLocal实例可访问的前提。而当线程消亡后，线程持有的这些本地变量副本都会被GC回收（除非还存在其他对变量副本的引用）。



### 重要变量和内部类

#### 内部类

**ThreadLocalMap**

ThreadLocalMap是专门为存放线程本地变量设计的哈希Map类，因此其所有方法都仅限于ThreadLocal类内部调用。

ThreadLocalMap的Entry使用弱引用关联ThreadLocal对象，并将该引用作为key。这样的设计有助于应对大对象存储以及线程生命周期长导致的内存占用问题。当ThreadLocal对象的强引用消亡后，其内存占用就会在之后的GC中回收。尽管如此，因为Entry并未使用reference queue来接收ThreadLocal对象回收通知，对应Entry只有在内部哈希表扩容时才能保证被移除。

另外，查看Thread类源码可以发现，其内部定义了一个ThreadLocalMap类型的域，这意味着每个线程都拥有自己用于存放线程本地变量的Map。

```java
static class ThreadLocalMap
```

***--Entry类***

Entry类继承自WeakReference(弱引用)类，并以ThreadLocal对象作为key。如果Entry.get返回null，意味着key的引用已经消亡，那么Entry就可以从Map中移除了。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

***--主要方法***

***----getEntry***

getEntry方法以指定ThreadLocal作为key从ThreadLocalMap中寻找对应的Entry。

搜索过程分为两步：首先使用key的哈希值算出Entry‘应该’存在的下标，如果在对应下标下找到指定Entry则直接返回，否则执行第二步，也就是调用getEntryAfterMiss方法从指定下标开始顺序查找对应元素。

*指定下标下不存在对应Entry的原因可能有两种：

1.指定Entry本来就不存在 2.因为哈希值碰撞导致指定Entry被放置到了其他位置。

![/myblog/images/ThreadLocal/getEntry.png](getEntry概念图)

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```