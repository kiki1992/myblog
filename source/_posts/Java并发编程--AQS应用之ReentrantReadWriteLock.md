---
title: Java并发编程--AQS应用之ReentrantReadWriteLock
date: 2018-02-05 14:28:32
summary: 本篇对Java并发框架中应用AQS实现计读写锁的ReentrantReadWriteLock源码进行了简单整理和记录。
tags: [多线程]
---

## 概要

> * ReentrantReadWriteLock类内部通过AQS实现了用于读写锁控制的Sync类，并在此基础上扩展了获取锁的公平/非公平策略，其内部读写锁类分别对应ReadLock和WriteLock。
>
> * 读写锁在公平策略下遵循以下规则：
>
>   ***读锁：***当一个线程试图获取读锁时（非重入的情况），如果存在其他线程持有写锁，或者有等待获取写锁的线程，则当前线程阻塞。另外，如果同步队列中有其他因为写线程而阻塞的读线程时，这些读线程也会优先获得锁。
>
>   ***写锁：***当一个线程试图获取写锁时（非重入的情况），如果存在其他线程持有读/写锁，或者有等待获取读/写锁的线程，则当前线程阻塞。
>
> * 读写锁在非公平策略下遵循以下规则：
>
>   ***读锁：*** 当一个线程试图获取读锁时（非重入的情况），如果存在其他线程持有写锁，则当前线程阻塞。
>
>   ***写锁：***当一个线程试图获取写锁时（非重入的情况），如果存在其他线程持有读/写锁，则当前线程阻塞。
>
> * 读写锁还存在以下区别：
>
>   写锁可以降级为读锁，只需要在持有锁期间获得读锁然后释放写锁即可。读锁无法升级成写锁。
>
>   写锁支持条件等待，而读锁则不支持。

## 主要原理

### 应用AQS实现的同步控制基础类Sync

#### 使用AQS的state变量同时维护读锁和写锁持有数量

ReentrantReadWriteLock内部使用AQS中定义的state变量同时维护了读锁和写锁的持有数量，即用一个int类型的变量的高16位和低16位来分别存储共享锁（读锁）和排它锁（写锁）的持有数量。

```java
static final int SHARED_SHIFT   = 16;
// 由于共享锁持有数量由state高16位维护，增减共享锁持有数量时需要加减65536而不是1。
static final int SHARED_UNIT    = (1 << SHARED_SHIFT); 
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** Returns the number of shared holds represented in count  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

#### 使用ThreadLocal保存线程持有读锁数量

ReentrantReadWriteLock中定义了ThreadLocal的子类来保存线程持有读锁的数量，这使得重入判断和监控线程持有的读锁数量变得方便。

```java
/**
 * A counter for per-thread read hold counts.
 * Maintained as a ThreadLocal; cached in cachedHoldCounter
 */
static final class HoldCounter {
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    final long tid = getThreadId(Thread.currentThread());
}

/**
 * ThreadLocal subclass. Easiest to explicitly define for sake
 * of deserialization mechanics.
 */
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

#### 应用了AQS排它锁以及共享锁获取/释放策略

Sync类同时应用了AQS中的排它锁以及共享锁获取/释放策略，来分别实现写锁/读锁方法逻辑。

以下是Sync类中实现的获取和释放排它锁以及共享锁的方法：

***排他锁***

```java
/** 释放排它锁 **/
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively()) // 当前持有锁的线程才能释放锁
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null); // 如果持有的锁全部释放则将线程独占标志清除
    setState(nextc);
    return free;
}

/** 获取排它锁 **/
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    // 有线程持有读/写锁
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // 有线程持有写锁但不是当前线程或是有其他线程持有写锁时获取排它锁失败
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 排它锁持有数量超过上限时抛出Error，否则重入锁
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    // 如果写锁需要阻塞或者CAS更新锁数量失败时获取排它锁失败
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

***共享锁***

```java
/** 释放共享锁 **/
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    // 检查并更新当前线程持有的共享锁数量
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    // 循环直至释放成功
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}

private IllegalMonitorStateException unmatchedUnlockException() {
    return new IllegalMonitorStateException(
        "attempt to unlock read lock, not locked by current thread");
}

/** 获取共享锁 **/
protected final int tryAcquireShared(int unused) {

    Thread current = Thread.currentThread();
    int c = getState();
    // 其他线程持有写锁，获取失败
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    // 读锁获取无需阻塞并且更新锁的持有数成功则获取成功
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    // 以上获取失败则采用死循环加CAS版本的获取方式
    return fullTryAcquireShared(current);
}

/** 获取共享锁 **/
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) {
            // 重入锁无论如何多不应该阻塞
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

另外，Sync中还定义了用于实现锁的公平/非公平获取策略的方法。

```java
/**
 * Returns true if the current thread, when trying to acquire
 * the read lock, and otherwise eligible to do so, should block
 * because of policy for overtaking other waiting threads.
 */
abstract boolean readerShouldBlock();

/**
 * Returns true if the current thread, when trying to acquire
 * the write lock, and otherwise eligible to do so, should block
 * because of policy for overtaking other waiting threads.
 */
abstract boolean writerShouldBlock();
```

### 基于Sync实现的公平锁/非公平锁

#### 公平锁FairSync

FairSync是基于Sync类实现的公平获取策略类，并扩展了readerShouldBlock/writerShouldBlock方法。

以下是FairSync类实现的获取许可方法：

```java
// 无论是读锁的获取还是写锁的获取，当同步队列中存在等待节点时当前线程都需要阻塞（非重入）
final boolean writerShouldBlock() {
    return hasQueuedPredecessors();
}
final boolean readerShouldBlock() {
    return hasQueuedPredecessors();
}
```

#### 非公平锁NonfairSync

NonfairSync是基于Sync类实现的非公平获取策略类，并扩展了readerShouldBlock/writerShouldBlock方法。

```java
// 获取写锁不应该阻塞
final boolean writerShouldBlock() {
    return false; // writers can always barge
}
// 获取读锁在同步队列中存在写锁等待时应该阻塞，这样可以尽量避免写操作‘饿死’
final boolean readerShouldBlock() {
    return apparentlyFirstQueuedIsExclusive();
}
```

## 总结及代码实践

ReentrantReadWriteLock通过AQS实现了读写锁分离的功能，内部同时使用了AQS中实现的排它锁和共享锁实现策略，尽管相比单独使用其中一种锁的实现略微复杂，但在理解AQS的实现原理之后还是比较容易理解的，下面就来看一个应用ReentrantReadWriteLock实现简单缓存的例子。

```java
public class CacheSample {

    private Object cache;
    private volatile boolean cacheAvailable;
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    /**
     * 缓存清理
     */
    public void clearCache() {

        writeLock.lock();
        try {
            cache = null;
            cacheAvailable = false;
            System.out.println("##CLR##current thread:" + Thread.currentThread().getName());
            System.out.println("cache cleared.");
        } finally {
            writeLock.unlock();
        }

    }

    /**
     * 获取数据
     */
    public Object getData() {

        // 首先需要获取读锁来检查cache是否生效
        readLock.lock();

        // 如果cache失效则需要释放读锁并获取写锁来更新cache
        try {
            if (!cacheAvailable) {
                readLock.unlock();
                writeLock.lock();
                try {
                    if (!cacheAvailable) {
                        cache = new Date(); // 这里只是举一个简单的例子
                        cacheAvailable = true;
                        System.out.println("##UPD##current thread:" + Thread.currentThread().getName());
                        System.out.println("cache updated. new cache:" + cache);
                    }
                    return cache;
                } finally {
                    writeLock.unlock();
                }

            }

        return cache;
        } finally {
            if (rwLock.getReadHoldCount() > 0) {
                readLock.unlock();
            }
        }

    }

}
```