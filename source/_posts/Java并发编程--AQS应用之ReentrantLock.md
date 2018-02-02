---
title: Java并发编程--AQS应用之ReentrantLock
date: 2018-02-01 11:08:32
summary: 本篇对Java并发框架中应用AQS实现可重入锁的ReentrantLock源码进行了简单整理和记录。
tags: [多线程]
---

## 概要

> * ReentrantLock类实现了一个可重入的互斥锁，其基本行为及语义同synchronized关键字一致，并在此基础上扩展了功能。
> * 一个ReentrantLock由最后成功获取锁并且尚未释放的线程持有。如果当前线程已经持有锁，那么调用获取锁的方法会立即返回。
> * ReentrantLock的构造方法支持指定锁的获取策略，如果指定为公平锁，获取锁的相关方法将会在发生争抢时保证等待时间最长的线程优先获得锁，否则锁的获取顺序将没有任何保证。
> * 通常情况下，将锁指定为公平锁时，大并发下系统的吞吐量将不如默认非公平策略，但同样会有更小的波动以及较少的饥饿情况。另外需要注意的是，公平的锁策略并不能帮助线程调度的公平性，而调用不带时间参数的tryLock方法时也会忽略公平策略。

## 主要原理

### 应用AQS实现的同步控制基础类Sync

Sync类利用AQS的state域变量来保存锁的重入次数，state等于0代表当前没有线程持有锁。当持有锁的线程再次执行获取锁的操作时state值会增加，而当线程执行释放锁的操作时state值会减少直至0则锁完全被释放。

以下是Sync类中实现的获取和释放锁的方法：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) { // 只有设置state成功才代表得到锁
            setExclusiveOwnerThread(current); // 获取到锁后需要设置当前拥有锁的线程
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { // 如果当前线程已经持有锁，只需要更新state值
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc); // 当前线程持有锁，更新操作无需同步
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread()) // 只有当前持有锁的线程才能执行释放锁的操作
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { // state归零表示锁已经完全释放。
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

另外Sync还支持条件等待，其实现则是直接利用了AQS中的ConditionObject。

```java
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

### 基于Sync实现的公平锁/非公平锁

#### 公平锁FairSync

FairSync是基于Sync类实现的公平锁，其释放锁的方法沿用了Sync中的实现，而获取锁的方法则使用了新的实现来保证公平策略。

以下是FairSync类实现的获取锁方法：

```java
final void lock() {
	acquire(1); //调用AQS中定义的acquire方法，具体获取锁的实现即tryAcquire方法
}

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) { // 当没有其他线程持有锁，且同步队列中没有前辈节点时才允许获取锁
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

#### 非公平锁NonfairSync

NonfairSync是基于Sync类实现的非公平锁，其获取锁的方式比较直接，如果设置state成功立即获得锁，否则在尝试获取锁的常规方式。

```java
final void lock() {
    if (compareAndSetState(0, 1)) // 设置state成功立即获得锁，无需考虑同步队列中是否有其他先驱线程。
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires); // 采用的是Sync中定义的非公平获取策略
}
```

## 总结及代码实践

如果理解了AQS的实现原理，那么ReentrantLock将非常容易理解，其实现的锁获取/释放，条件等待基本都复用了AQS中定义的方法。

最后，我们来看一个使用ReentrantLock实现同步的简单例子。

```java
public class ObjQueue {

    private int size, getIndex, putIndex;
    private int capacity;
    private Object[] objs;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition(); // 等待条件：队列未满
    private final Condition notEmpty = lock.newCondition(); // 等待条件：队列非空

    public ObjQueue(int capacity) {
        this.capacity = capacity;
        objs = new Object[capacity];
    }

    // 取出对象的方法
    public Object getObj() throws InterruptedException {

        lock.lock();
        try {

            while(size == 0) {
                System.out.println("Thread " + Thread.currentThread().getName() + " waiting for put...");
                notEmpty.await();
            }

            Object obj = objs[getIndex];
            if (++getIndex >= capacity) {
                getIndex = 0;
            }
            size--;
            System.out.println("Thread " + Thread.currentThread().getName() + " get " + obj);
            notFull.signal();
            return obj;
        } finally {
            lock.unlock();
        }

    }

    // 放入对象的方法
    public void putObj(Object obj) throws InterruptedException {

        lock.lock();
        try {
            while (size == capacity) {
                System.out.println("Thread " + Thread.currentThread().getName() + " waiting for get...");
                notFull.await();
            }
            objs[putIndex] = obj;
            if (++putIndex >= capacity) {
                putIndex = 0;
            }
            size++;
            System.out.println("Thread " + Thread.currentThread().getName() + " put " + obj);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }

    }


    public static void main(String[] args) {

        ObjQueue objQueue = new ObjQueue(10);
        
        ExecutorService executorServicePut = Executors.newFixedThreadPool(10);
        for (int count = 0; count < 100; count++) {
            executorServicePut.submit(() -> {
                try {
                    objQueue.putObj(new Object());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        
        ExecutorService executorServiceGet = Executors.newFixedThreadPool(10);
        for (int count = 0; count < 100; count++) {
            executorServiceGet.submit(() -> {
                try {
                    Thread.sleep(2000);
                    objQueue.getObj();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        executorServiceGet.shutdown();
        executorServicePut.shutdown();

    }

}

```