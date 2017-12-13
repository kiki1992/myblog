---
title: CopyOnWriteArrayList源码笔记
date: 2017-12-13 11:06:32
summary: 本篇对CopyOnWriteArrayList源码做了简单分析及整理
tags: [Java Collections Framework,多线程]
---

# CopyOnWriteArrayList源码笔记
```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable

```
## 主要特性
> * 1.本类是ArrayList的线程安全版本之一，但其内部实现不存在ArrayList中类似的扩容机制)，所有修改和遍历操作都基于一个底层数组的最新拷贝(当前状态的快照)来实现。
> * 2.使用本类应该注意以下几点:<br>
>		 （1 当遍历操作远多于修改操作，并且需要存放的数据不会很多时才考虑使用本类，原因是修改操作中的数组拷贝可能造成巨大的时间和空间消耗。-- 常见的例子是用于注册监听器列表    
>		 （2 当不要求数据实时一致性时才考虑使用本类，原因是修改操作中的数组拷贝，元素增删都需要时间，在执行修改操作后读操作获取的可能还是修改前的数据。
> * 3.本类的内存一致性效果:在一个线程将对象A放入CopyOnWriteArrayList的操作happens-before其它线程接下来访问或者移除对象A的操作。

## 重要域变量
```java
/** The lock protecting all mutators */
final transient ReentrantLock lock = new ReentrantLock(); // 修改操作使用的锁

/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array; // 后台维护的数组，只能通过以下方法访问，以保证数据的一致性

// 可以访问array的方法
// ***注意***：getArray方法取得的只是当前状态的快照
final Object[] getArray() {
    return array;
}
final void setArray(Object[] a) {
    array = a;
}
```

## 构造器
CopyOnWriteArrayList类提供了以下三种构造器方法，在使用上并无特殊的地方。
```java
// 创建一个空的列表
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
// 创建一个包含指定集合内所有元素的列表
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
// 创建一个存放指定数组元素拷贝的列表
public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
```

## 主要方法
#### 读取操作
常用的读取操作方法并没有太复杂的逻辑，因此这里只是做了一个简单的罗列。
不过我们可以在这些方法里找到一些共性，即所有方法都是基于getArray方法实现的，这也和上面提到的array只能通过getArray/setArray方法访问的说法不谋而合。
鉴于所有读取操作方法都没有做同步，因此每次读取到的数据都只是当前列表状态的一个快照。
```java
// 获取列表元素数量
public int size() {
    return getArray().length;
}
// 判断列表是否为空
public boolean isEmpty() {
    return size() == 0;
}
// 判断列表是否包含指定元素
public boolean contains(Object o) {
    Object[] elements = getArray();
    return indexOf(o, elements, 0, elements.length) >= 0;
}
// 获取指定元素在列表中第一次出现的下标
public int indexOf(Object o) {
    Object[] elements = getArray();
    return indexOf(o, elements, 0, elements.length);
}
// 从指定下标开始正向寻找元素
public int indexOf(E e, int index) {
    Object[] elements = getArray();
    return indexOf(e, elements, index, elements.length);
}
// 获取指定元素在列表中最后一次出现的下标
public int lastIndexOf(Object o) {
    Object[] elements = getArray();
    return lastIndexOf(o, elements, elements.length - 1);
}
// 从指定下标开始反向寻找元素
public int lastIndexOf(E e, int index) {
    Object[] elements = getArray();
    return lastIndexOf(e, elements, index);
}
// 将列表转换为数组
public Object[] toArray() {
    Object[] elements = getArray();
    return Arrays.copyOf(elements, elements.length);
}
// 将列表转换为数组 -- 如果指定数组a容量够大直接使用a存放元素，否则另外开辟内存存放元素
public <T> T[] toArray(T a[]) {
    Object[] elements = getArray();
    int len = elements.length;
    if (a.length < len)
        return (T[]) Arrays.copyOf(elements, len, a.getClass());
    else {
        System.arraycopy(elements, 0, a, 0, len);
        if (a.length > len)
            a[len] = null;
        return a;
    }
}
// 获取列表指定下标位置的元素
public E get(int index) {
    return get(getArray(), index);
}
```

#### 修改操作
修改操作相关方法代码比较多就不全贴了，具体可以参考jdk源码。
这里只记录所有修改方法共同的处理流程，主要有以下两种方式：
获取元数组->拷贝元数组->基于拷贝修改->修改后的拷贝作为新数组
或者是
获取元数组->使用临时数组存放基于元数组元素处理后的数据->根据临时数组设置新数组
下面分别是两种处理方式的例子
```java
// 新增元素 -- 第一种方式
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 获取元数组
        Object[] elements = getArray();
        int len = elements.length;
        // 拷贝元数组(长度+1)
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 将指定元素加入数组拷贝
        newElements[len] = e;
        // 设置拷贝作为新数组
        setArray(newElements);
        return true;
    } finally {
    	// 释放锁
        lock.unlock();
    }
}

// 根据指定过滤器移除元素 -- 第二种方式
public boolean removeIf(Predicate<? super E> filter) {
    if (filter == null) throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
    	// 获取元数组
        Object[] elements = getArray();
        int len = elements.length;
        if (len != 0) {
            int newlen = 0;
            // 使用临时数组存放满足对应条件的元素
            Object[] temp = new Object[len];
            for (int i = 0; i < len; ++i) {
                @SuppressWarnings("unchecked") E e = (E) elements[i];
                if (!filter.test(e))
                    temp[newlen++] = e;
            }
            // 根据临时数组设置新数组
            if (newlen != len) {
                setArray(Arrays.copyOf(temp, newlen));
                return true;
            }
        }
        return false;
    } finally {
    	// 释放锁
        lock.unlock();
    }
}
```
从CopyOnWriteArrayList类的修改操作共性也可以看出，所有修改操作都会造成空间上的额外消耗，无论是拷贝数组开始临时数组的使用。因此在决定使用本类前最好对元素的数量/大小有一个恰当的评估。

#### 迭代操作
CopyOnWriteArrayList类内部实现了基于ListIterator的迭代器类COWIterator。
```java
static final class COWIterator<E> implements ListIterator<E>
```
从其定义的域变量可以看到，COWIterator使用其创建时列表内部数组的一个快照执行遍历操作。
```java
/** Snapshot of the array */
private final Object[] snapshot; // 数组快照
/** Index of element to be returned by subsequent call to next.  */
private int cursor; // 下一个元素的下标

// 例如next方法是这样实现的
public E next() {
    if (! hasNext())
        throw new NoSuchElementException();
    return (E) snapshot[cursor++];
}
```
另外，COWIterator不支持remove,set以及add操作，调用这些操作会导致抛出UnsupportedOperationException。例如remove方法的实现如下：
```java
public void remove() {
    throw new UnsupportedOperationException();
}
```
从以上说明可以看出，使用CopyOnWriteArrayList类的迭代器进行遍历时不需要额外的同步处理，调用其任何方法也不会导致ArrayList迭代器方法中ConcurrentModificationException的发生(迭代器本身不支持对数组结构的修改，同时其它通过迭代器以外方法修改数组结构的操作也不会反映在迭代器的数组快照上)。这样做既有好处也有坏处，好处是不需要额外的同步操作开销，坏处也很明显，不能实时保证数据的一致性，迭代过程中元数据随时可能遭到修改。

#### 子列表
CopyOnWriteArrayList内部实现了基于AbstractList的列表类COWSubList。
```java
private static class COWSubList<E>
    extends AbstractList<E>
    implements RandomAccess
```
其内部维护了以下变量：
```java
private final CopyOnWriteArrayList<E> l; // 父列表
private final int offset; // 从父列表开始的下标
private int size; // 子列表长度
private Object[] expectedArray; // 同步检测数组
```
与CopyOnWriteArrayList不同的是，COWSubList类对读取操作也进行了同步操作，并且在所有操作中都会检查上述expectedArray是否与当前父列表的内部数组一致，不一致说明父列表内部数组结构修改未经过当前子列表方法，会导致ConcurrentModificationException抛出。比如下面的get方法：
```java
public E get(int index) {
    final ReentrantLock lock = l.lock;
    // 获取锁
    lock.lock();
    try {
        rangeCheck(index);
        // 数组结构修改检查
        checkForComodification();
        return l.get(index+offset);
    } finally {
    	// 释放锁
        lock.unlock();
    }
}
```
需要注意的是，并非所有方法都进行了同步操作，比如forEach。
forEach方法会基于expectedArray数组的一个快照进行遍历操作，因为没有其它操作会改变这个快照的地址，所以遍历操作是线程安全的。
```java
public void forEach(Consumer<? super E> action) {
    if (action == null) throw new NullPointerException();
    int lo = offset;
    int hi = offset + size;
    Object[] a = expectedArray;
    if (l.getArray() != a)
        throw new ConcurrentModificationException();
    if (lo < 0 || hi > a.length)
        throw new IndexOutOfBoundsException();
    for (int i = lo; i < hi; ++i) {
        @SuppressWarnings("unchecked") E e = (E) a[i];
        action.accept(e);
    }
}
```


