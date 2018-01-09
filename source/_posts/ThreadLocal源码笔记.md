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



***

### 重要变量和内部类

#### 重要变量 

```java
// 标记当前ThreadLocal对象的哈希值
private final int threadLocalHashCode = nextHashCode();

// 用于保证线程安全的AtomicInteger变量
private static AtomicInteger nextHashCode =
    new AtomicInteger();

// 哈希值递增规则，使其尽量分散
private static final int HASH_INCREMENT = 0x61c88647;
```

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

![getEntry概念图](/myblog/images/ThreadLocal/getEntry.png)

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

***----getEntryAfterMiss***

getEntryAfterMiss方法从指定下标开始顺序搜索指定key对应的Entry直到遇到Entry为null或者找到对应Entry为止。如果遍历查找过程中发现Entry的key为null，即Entry不再被引用，则会触发清除操作。
![getEntryAfterMiss概念图](/myblog/images/ThreadLocal/getEntryAfterMiss.png)
```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key) // 找到指定key对应的Entry，返回
            return e;
        if (k == null) // 发现key为null的Entry，移除
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len); // 对比下一个Entry
        e = tab[i];
    }
    return null;
}
```

**小结Tips：**

getEntry方法由ThreadLocal的get方法调用，从上面的源码分析可以得知，当调用ThreadLocal的get方法时，实际会以ThreadLocal作为key在当前线程上的ThreadLocalMap寻找对应的Entry，而寻找的过程中可能还伴随一些***移除无用Entry***的处理。

***----set***

set方法将从由指定key的哈希值计算出的下标开始寻找可以放置key/value的位置。这个过程可能涉及到value的替换/Entry的替换/扩容等操作。
![set概念图](/myblog/images/ThreadLocal/setThreadLocalMap.png)
```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    // 从指定下标开始寻找可以放置指定key/value的位置，搜索过程遵循以下规则：
    // 1.发现和指定key一致的Entry，替换value后返回
    // 2.发现key为null的Entry，替换改Entry并返回
    // 3.发现可能放置Entry的空位，放置key/value后检查是否需要扩容
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 首先尝试清除一些slot，如果没有slot可以清除并且达到扩容阈值则执行扩容操作
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

***----replaceStaleEntry***

replaceStaleEntry方法会替换指定无用Entry，替换过程可能无法发现指定key对应的Entry已经存在，这会导致同一key对应的Entry存在多个。

另外，调用本方法还将清除包含指定无用Entry区间内的所有其他无用Entry(区间以两个空闲slot作为开始和结束)。
![replaceStaleEntry概念图](/myblog/images/ThreadLocal/replaceStaleEntry.png)
```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // 首先找到包含指定staleSlot区间内的第一个staleSlot
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // 从指定stateSlot的位置开始遍历，直到出现指定key对应的Entry/空闲slot
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 如果发现指定key对应的Entry，
        // 将指定stateSlot的Entry和当前位置的Entry交换后执行清理工作
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 指定key对应的Entry未找到，则直接替换stateSlot下的Entry
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 如果包含stateSlot的区间内存在无用Entry，则执行清理工作
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

**小结Tips：**

set方法由ThreadLocal的set/setInitialValue方法调用，在利用ThreadLocal设值时可能会出现指定key对应的Entry重复的现象。设值过程也可能触发Slot清理，扩容等操作。

***----remove***

remove方法将移除指定key对应的Entry并执行一些清理工作。

```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // 根据key的哈希值算出Entry下标，从此下标开始寻找指定key对应的Entry并移除
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

***----expungeStaleEntry***

expungeStaleEntry方法在清除指定stateSlot下标开始至下一空闲slot区间内无用Entry的同时，尝试重新计算区间内各个Entry key的哈希值并将Entry移动至合适的位置。

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 检测到的staleEntry置null，方便GC
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // 将指定staleSlot->下一空闲slot区间内的所有staleEntry置null
    // 并重新调整有效Entry的位置
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 检测到的staleEntry置null，方便GC
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                // 将Entry放置到哈希值重新计算后的位置，如果已经被占用，寻找下一个空闲slot
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

***----cleanSomeSlots***

cleanSomeSlots方法会从指定位置开始查找无用Entry并将其移除。查找的次数会被控制在对数级别，以在垃圾清理数量和时间消耗之间取得一个平衡点。

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    // 默认从指定位置开始检查最多log2(n)个slot，发现无用Entry则移除
    // 如果查找过程中发现无用Entry，则最多检查log2(Entry个数)个slot
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

***----rehash/resize***

rehash/resize方法用于检查和执行扩容操作。

```java
private void rehash() {
    expungeStaleEntries();

    // 如果当前Entry个数达到扩容阈值的3/4则执行扩容
    if (size >= threshold - threshold / 4)
        resize();
}
```

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
    
    // 将扩容前哈希表中的Entry转移至扩容后新创建的哈希表中。
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // 遍历旧哈希表的过程中发现无用Entry将其value置null帮助GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```
***

### 主要方法

***initialValue***

如果需要在第一次执行ThreadLocal的get方法时返回一个初始值，那么就可以重写本方法。不过在使用本方法时需要注意一下两点：

1.如果在调用get方法之前已经调用了set方法，则本方法将不会再被调用。

2.本方法最多被调用一次。除非执行get方法后又执行了remove方法。

```java
// 默认实现直接返回null
protected T initialValue() {
    return null;
}
```

***get***

get方法将以ThrealLocal作为key从当前线程的ThreadLocalMap中获取指定线程本地变量值。如果找不到对应变量值，则通过initialValue方法取得初始值。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    // 寻找指定线程本地变量
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 上述处理找不到则设置并返回初始值
    return setInitialValue();
}
```

***setInitialValue***

setInitialValue方法将通过initialValue方法获取初始值并将其设置到当前线程的ThreadLocalMap中，如果Map还未创建就新创建。这意味着线程本地变量Map是***延迟加载***的。

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

```

***set***

set方法将指定value设置到当前线程的ThreadLocalMap中，其中key是ThreadLocal对象引用。如果Map还未创建就新创建。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

***remove***

remove方法将把当前ThreadLocal对应的线程本地变量从线程ThreadLocalMap中移除。

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

