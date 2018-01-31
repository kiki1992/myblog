---
title: Java并发编程--AQS源码笔记
date: 2018-01-11 15:17:32
summary: 本篇对Java并发框架中的AQS(AbstractQueuedSynchronizer)源码做了简单的整理和记录。
tags: [多线程]
---

## AQS源码笔记

### 所有权获取/释放

#### 独占模式

##### -- 所有权获取 --

***acquire -- 所有权获取入口***

acquire作为所有权获取的入口，可以用来实现获取锁的方法。

```java
public final void acquire(int arg) {
    // 方法首先尝试获取所有权，如果失败当前线程将被放入内部同步等待队列。
    // acquireQueued方法中线程将重复阻塞/非阻塞的切换，直到最终获得所有权。
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

***acquireInterruptibly -- 所有权获取入口(可中断)***

本方法和acquire的不同之处在于其在收到中断信号时将中止等待。

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    // 方法将首先检查线程的中断状态，如果中断状态被设置则直接抛出异常，否则尝试获取所有权。
    // doAcquireInterruptibly方法中线程将重复阻塞/非阻塞的切换，直到最终获得所有权或者收到中断信号。
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

***tryAcquireNanos ---  所有权获取入口(可中断+带超时)***

本方法和acquireInterruptibly的不同之处在于其等待带有时间限制。

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    // 方法将首先检查线程的中断状态，如果中断状态被设置则直接抛出异常，否则尝试获取所有权。
    // doAcquireInterruptibly方法中线程将重复阻塞/非阻塞的切换，直到最终获得所有权/收到中断信号/等待超时。
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

***acquireQueued -- 同步队列节点尝试获取所有权***

本方法采用死循环+阻塞的方式为同步队列中的节点获取所有权提供服务。节点代表的线程会不断重复检查所有权获取/阻塞的过程，直至成功获取到所有权。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

***doAcquireInterruptibly -- 同步队列节点尝试获取所有权（可中断）***

本方法和acquireQueued的区别就是线程阻塞状态下收到中断信号将终止等待。

```java
// 重点代码
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    throw new InterruptedException(); // 中断
```

***doAcquireNanos -- 同步队列节点尝试获取所有权（可中断+带超时）***
本方法和doAcquireInterruptibly的区别就是线程等待时间是有限的。
```java
// 重点代码
nanosTimeout = deadline - System.nanoTime();
if (nanosTimeout <= 0L) // 等待超时
    return false;
if (shouldParkAfterFailedAcquire(p, node) &&
    nanosTimeout > spinForTimeoutThreshold) // 如果等待时间过短，采用自旋转方式比阻塞更有效
    LockSupport.parkNanos(this, nanosTimeout);
if (Thread.interrupted())
    throw new InterruptedException(); // 中断
```

***tryAcquire -- 尝试获取所有权***

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

独占模式下需要使用tryAcquire方法来尝试获取所有权，如果获取失败，那么执行本方法的线程可能会被放入内部的FIFO同步队列，前提是线程尚未入列。之后便需要等待其他线程发送释放所有权的信号。

AQS并未对tryAcquire方法做默认实现，而是抛出UnsupportedOperationException异常。当我们在实现本方法时可以重点关注以下两点点：

* 真正执行尝试获取所有权的操作之前，先检查当前AQS状态是否允许在独占模式下获取所有权。
* 对公平锁/非公平锁的实现可以通过本方法完成，举例来说，ReentrantLock中的公平锁实现就在本方法中对同步队列状态做了检查，当有新线程尝试获取锁时首先会判断同步队列中是否还有等待中的线程，如果有则获取失败。另外，如果需要实现其他规则，也可以通过本方法来自定义。

##### -- 所有权释放 --

***release -- 释放所有权***

释放所有权的操作会在tryRelease方法返回true时操作，并触发对后继线程的唤醒操作。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

***tryRelease -- 尝试释放所有权***

```java
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```

本方法用于独占模式下的所有权释放操作。默认实现不支持操作。

#### 共享模式

##### -- 所有权获取 --

***acquireShared -- 共享模式下所有权获取入口***

本方法作为共享模式下所有权获取的入口。

```java
public final void acquireShared(int arg) {
  // 如果获取所有权失败，线程将被放入同步等待队列，并重复阻塞/解除状态直至所有权获取成功。  
  if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```
***acquireSharedInterruptibly -- 共享模式下所有权获取入口(可中断)***

本方法和acquireShared的区别在于可中断。

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

***tryAcquireSharedNanos -- 共享模式下所有权获取入口(可中断+带超时)***

本方法和acquireSharedInterruptibly的区别在于带超时。

```java
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}
```

***doAcquireShared -- 共享模式下同步队列节点获取所有权***

本方法和独占模式下的doAcquire方法区别在于获取所有权成功之后可能唤醒后继节点并导致唤醒操作传播。

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 设置head并传播后继节点唤醒操作（r>0的情况下才会执行唤醒）
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
***doAcquireSharedInterruptibly -- 共享模式下同步队列节点获取所有权(可中断)***

本方法和doAcquireShared的区别在于收到中断信号时将中止等待，抛出异常。

```java
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    throw new InterruptedException();
```

***doAcquireSharedNanos -- 共享模式下同步队列节点获取所有权(可中断+带超时)***

本方法和doAcquireSharedInterruptibly的区别在于超时也将终止等待。

```java
nanosTimeout = deadline - System.nanoTime();
if (nanosTimeout <= 0L)
    return false;
if (shouldParkAfterFailedAcquire(p, node) &&
    nanosTimeout > spinForTimeoutThreshold)
    LockSupport.parkNanos(this, nanosTimeout);
```

***tryAcquireShared -- 共享模式下尝试获取所有权***

共享模式下需要使用tryAcquireShared 方法来尝试获取所有权，如果获取失败，那么执行本方法的线程可能会被放入内部的FIFO同步队列，前提是线程尚未入列。之后便需要等待其他线程发送释放所有权的信号。

AQS并未对tryAcquire方法做默认实现，而是抛出UnsupportedOperationException异常。本方法的返回值有特殊含义，需要特别注意，以下是对应关系：

1.负数：所有权获取失败

2.零：当前获取操作成功，但后继节点无法成功获取

3.正数：当前操作获取成功，且后继节点也可能成功获取

***将返回值分为三类使本方法可以适用于某些需要独占获取所有权的场景***

##### -- 所有权释放 --

***releaseShared -- 共享模式下释放所有权***

releaseShared方法会在tryReleaseShared返回true的情况下触发后续节点的唤醒或者head节点传播状态设置。

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

***doReleaseShared -- 后继节点唤醒或传播状态设置***

本方法用于后续节点的唤醒或者head节点传播状态设置。

```java
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 如果后继节点正在等待唤醒信号，则执行唤醒操作
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 如果没有后继节点或者后继节点尚未阻塞，则设置传播状态
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

***tryReleaseShared --共享模式下尝试获取所有权***

```java
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```

### 同步等待队列相关操作

#### 入列

***addWaiter -- 新增等待线程入列***

addWaiter方法以指定模式(独占/共享)将当前线程包装成等待节点并放入同步等待队列。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 在调用enq方法入列之前首先采用更快的入列方式(假设队列已经初始化)
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

***enq -- 等待线程节点入列***

enq方法将指定节点放入同步等待队列，除了在新增等待线程节点时调用，本方法在使用Condition时还用于将指定节点从condition队列移动至同步等待队列。

当入列操作发生竞争时，会不断重试直至入列成功。

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        // 如果队列尚未初始化则执行初始化
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            // cas操作执行入列操作
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```



### 线程阻塞/唤醒相关操作

***shouldParkAfterFailedAcquire -- 判断线程是否应该在所有权获取失败后阻塞***

本方法在线程获取所有权失败后调用，综合当前节点的前辈节点状态判断线程需要阻塞还是重试。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 已经设置前辈节点的waitStatus为SIGNAL，当前线程可以直接阻塞。
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    // 直属前辈节点已经取消，则向前寻找最近未取消的前辈节点。
    // 此后调用方再根据最新的前辈节点重试之前的操作。
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    // 尚未设置前辈节点的waitStatus为SIGNAL，则设置其waitStatus，但线程还不用阻塞。
    // 此后调用方还有一次机会确认是否可以在阻塞前获得所有权。
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

***parkAndCheckInterrupt --- 执行线程阻塞***

本方法利用LockSupport的park使当前线程阻塞，直至收到中断信号或者由前辈节点唤醒，并返回线程的中断状态。

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```
***unparkSuccessor -- 唤醒同步队列后继节点***

本方法用于唤醒同步等待队列中阻塞的后继节点。

**调用场景**：1.当前节点取消 2.出列节点释放所有权

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    // 如果遇到直属后继节点取消的情况，需要从队列尾部开始遍历最近的后继节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
***setHeadAndPropagate -- 唤醒传播***

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    // 如果tryAcquireShared方法调用返回值>0（后继节点有机会获取所有权）
    // 或者之前的操作设置了需要传播，并且后继节点为共享模式，则执行后继线程的唤醒
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared(); // doReleaseShared可能执行后继线程的唤醒或者传播标示的设置
    }
}
```

### 等待条件

> * AQS中的等待条件类ConditionObject内部维护了一个条件等待队列，当调用await相关方法时线程会被包装成等待Node并放入队列，并在其他线程调用signal或signalAll方法时从等待队列中转移至AQS类中维护的同步等待队列中。

#### 调用await相关方法使线程进入条件等待

***awaitUninterruptibly -- 不可中断条件等待***

调用awaitUninterruptibly使线程释放所有权并进入条件等待，直至收到signal信号。

```java
public final void awaitUninterruptibly() {
    Node node = addConditionWaiter(); // 将线程放入条件等待队列
    int savedState = fullyRelease(node); // 释放所有权
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) { // 等待直到线程被移动至AQS同步队列并且被唤醒
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true;
    }
    // 唤醒后尝试获得所有权
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}
```

***await -- 可中断条件等待***

调用await使线程释放所有权并进入条件等待，直至收到signal信号或者被中断。如果线程在收到signal信号之前被中断将抛出InterruptedException，而如果是在收到信号之后中断则设置线程的中断状态。

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter(); // 将线程放入条件等待队列
    int savedState = fullyRelease(node); // 释放所有权
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) { // 等待直到线程被移动至AQS同步队列并且被唤醒
        LockSupport.park(this);
        // 检查等待过程中的中断状态，如果中断状态被设置则终止条件等待
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters(); // 如果是收到signal信号正常返回，nextWaiter会被清空
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

***awaitNanos/awaitUntil/await(long time, TimeUnit unit) -- 带超时条件等待***

以上三个方法和await的区别就是多了超时等待。

#### 调用signal相关方法发送信号

***signal -- 单一通知***

调用signal方法将条件队列中等待时间最久的未取消线程移动至AQS同步队列中。

```java
public final void signal() {
    if (!isHeldExclusively()) // 本方法调用成功的前提是当前线程持有所有权
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
    	if ( (firstWaiter = first.nextWaiter) == null)
    		lastWaiter = null;
    	first.nextWaiter = null;
    } while (!transferForSignal(first) && // transferForSignal方法将执行队列节点的移动
    	(first = firstWaiter) != null);
}
```
***signalAll -- 全体通知***

调用signalAll 方法将条件队列中所有未取消线程移动至AQS同步队列中。

```java
public final void signalAll() {
    if (!isHeldExclusively()) // 本方法调用成功的前提是当前线程持有所有权
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}
private void doSignalAll(Node first) {
	lastWaiter = firstWaiter = null;
	do {
		Node next = first.nextWaiter;
		first.nextWaiter = null;
		transferForSignal(first); // transferForSignal方法将执行队列节点的移动
		first = next;
	} while (first != null);
}
```

