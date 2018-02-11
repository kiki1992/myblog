### 写锁获取

#### writeLock

获取独占写锁时，首先会判断当前是否有其他线程持有独占写锁/共享读锁，如果没有则尝试直接获取锁，否则将调用acquireWrite方法触发锁获取自旋，阻塞等一系列事件。

acquireWrite方法是StampedLock中定义的相对复杂的写锁获取方法，将在下面做详细的分析和记录。

```java
public long writeLock() {
    long s, next;  // bypass acquireWrite in fully unlocked case only
    return ((((s = state) & ABITS) == 0L &&
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
            next : acquireWrite(false, 0L));
}
```

#### tryWriteLock 

不带参版本的tryWriteLock方法和writeLock的不同之处在于前者会在尝试获取写锁失败后立即返回，而不会调用acquireWrite方法。

```java
public long tryWriteLock() {
    long s, next;
    return ((((s = state) & ABITS) == 0L &&
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
            next : 0L);
}
```

另外，tryWriteLock还有一个带超时时间参数的版本，相比writeLock多了超时等待和可中断特性。

```java
public long tryWriteLock(long time, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(time);
    if (!Thread.interrupted()) {
        long next, deadline;
        if ((next = tryWriteLock()) != 0L)
            return next;
        if (nanos <= 0L)
            return 0L;
        if ((deadline = System.nanoTime() + nanos) == 0L)
            deadline = 1L;
        if ((next = acquireWrite(true, deadline)) != INTERRUPTED)
            return next;
    }
    throw new InterruptedException();
}
```

#### writeLockInterruptibly

本方法和writeLock的区别在于拥有可中断特性。

```java
public long writeLockInterruptibly() throws InterruptedException {
    long next;
    if (!Thread.interrupted() &&
        (next = acquireWrite(true, 0L)) != INTERRUPTED)
        return next;
    throw new InterruptedException();
}
```

#### acquireWrite

下面将acquireWrite方法分成两部分进行分析和说明。

***等待线程enqueue前自旋过程***

每个线程enqueue前自旋的过程：

1.尝试获取锁，成功则返回更新后的state值，否则进入自旋判断

2.线程首次进入将根据当前队列及锁的获取情况判断是否需要自旋地获取锁，如果需要则开始自旋，否则进入enqueue操作

3.执行enqueue操作的主要步骤如下：

* 队列未初始化则尝试初始化，并重试获取锁
* 线程未包装成等待节点则执行包装，并重试获取锁
* 等待节点前驱节点发生变化则更新节点，并重试获取锁
* 尝试将等待节点enqueue，成功则退出循环，否则重试获取锁

```java
for (int spins = -1;;) { // spin while enqueuing
    long m, s, ns;
    // enqueue前再次确认是否能够获得锁
    if ((m = (s = state) & ABITS) == 0L) {
        if (U.compareAndSwapLong(this, STATE, s, ns = s + WBIT))
            return ns;
    }
    // 判断当前线程在enqueue前是否需要自旋的依据：
    // 当前没有其他线程获得读锁并且当前线程如果enqueue可能成为队列头结点
    else if (spins < 0)
        spins = (m == WBIT && wtail == whead) ? SPINS : 0;
    else if (spins > 0) {
        if (LockSupport.nextSecondarySeed() >= 0)
            --spins;
    }
    // 如果等待队列没有初始化，则尝试初始化
    else if ((p = wtail) == null) { // initialize queue
        WNode hd = new WNode(WMODE, null);
        // 由于多线程竞争，队列可能由其他线程初始化
        if (U.compareAndSwapObject(this, WHEAD, null, hd))
            wtail = hd;
    }
    // 当前线程未包装成等待节点，则执行包装
    else if (node == null)
        node = new WNode(WMODE, p);
    // 队列尾部节点随时可能更新，这里需要判断尾部节点是否变化
    else if (node.prev != p)
        node.prev = p;
    // 真正尝试将节点enqueue，成功则退出循环
    else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
        p.next = node;
        break;
    }
}
```

***enqueue后队列头自旋***

```java
for (int spins = -1;;) {
    WNode h, np, pp; int ps;
    // 如果当前节点的前驱节点是头结点（头结点非等待线程），则节点开始自旋地获取锁
    // 头结点自旋具有一个初始自旋次数，每次完成整个自旋过程后若线程还不能获得锁，次数会不断增多直到到达
    // 上限次数，之后每个自旋过程的次数将维持在上限次数。
    if ((h = whead) == p) {
        if (spins < 0)
            spins = HEAD_SPINS;
        else if (spins < MAX_HEAD_SPINS)
            spins <<= 1;
        // 头节点自旋过程，每次的自旋次数随着时间推移递增
        for (int k = spins;;) { // spin at head
            long s, ns;
            if (((s = state) & ABITS) == 0L) {
                if (U.compareAndSwapLong(this, STATE, s,
                                         ns = s + WBIT)) {
                    whead = node;
                    node.prev = null;
                    return ns;
                }
            }
            else if (LockSupport.nextSecondarySeed() >= 0 &&
                     --k <= 0)
                break;
        }
    }
    // TODO
    else if (h != null) { // help release stale waiters
        WNode c; Thread w;
        while ((c = h.cowait) != null) {
            if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                (w = c.thread) != null)
                U.unpark(w);
        }
    }
    // 等待节点进入阻塞的过程
    // 如果进入阻塞前队列情况发生变化，比如头结点变了，当前节点的前驱节点取消了等等，都意味着当前节点可能
    // 成为下一个自旋节点，因此需要在最终阻塞前在回到开头判断当前队列的实际状态。
    if (whead == h) {
        // 前驱节点发生变化
        if ((np = node.prev) != p) {
            if (np != null)
                (p = np).next = node;   // stale
        }
        // 前驱节点还未设置等待状态 TODO：等待状态可能在唤醒操作时使用
        else if ((ps = p.status) == 0)
            U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
        // 前驱节点已经取消，需要更新节点信息
        else if (ps == CANCELLED) {
            if ((pp = p.prev) != null) {
                node.prev = pp;
                pp.next = node;
            }
        }
        // 队列状态暂时未检测到变化，准备进入阻塞
        else {
            // 超时等待的一些判断
            long time; // 0 argument to park means no timeout
            if (deadline == 0L)
                time = 0L;
            else if ((time = deadline - System.nanoTime()) <= 0L)
                return cancelWaiter(node, node, false);
            Thread wt = Thread.currentThread();
            U.putObject(wt, PARKBLOCKER, this);
            node.thread = wt;
            // 真正阻塞前再一次判断队列状态是否变化，还是没有变化则真正阻塞
            if (p.status < 0 && (p != h || (state & ABITS) != 0L) &&
                whead == h && node.prev == p)
                U.park(false, time);  // emulate LockSupport.park
            // 节点被唤醒或者被中断
            node.thread = null;
            U.putObject(wt, PARKBLOCKER, null);
            if (interruptible && Thread.interrupted())
                return cancelWaiter(node, node, true);
        }
    }
}
```

### 读锁获取

#### readLock

获取共享读锁时，如果不存在竞争的情况则线程将直接尝试获取锁，如果失败再调用acquireRead方法触发锁获取自旋，阻塞等一系列事件。

```java
public long readLock() {
    long s = state, next;  // bypass acquireRead on common uncontended case
    // 判断无竞争的条件是等待队列为空以及持有读锁的线程数量未超过RFULL
    return ((whead == wtail && (s & ABITS) < RFULL &&
             U.compareAndSwapLong(this, STATE, s, next = s + RUNIT)) ?
            next : acquireRead(false, 0L));
}
```

#### tryReadLock

不带参版本的tryReadLock方法会在锁可用的情况下尝试立即获取，失败则立即返回。

```java
public long tryReadLock() {
    for (;;) {
        long s, m, next;
        if ((m = (s = state) & ABITS) == WBIT)
            return 0L;
        else if (m < RFULL) {
            if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                return next;
        }
        else if ((next = tryIncReaderOverflow(s)) != 0L)
            return next;
    }
}
```

另外，带超时参数的tryReadLock版本相比readLock方法增加了超时等待和可中断特性。

```java
public long tryReadLock(long time, TimeUnit unit)
    throws InterruptedException {
    long s, m, next, deadline;
    long nanos = unit.toNanos(time);
    if (!Thread.interrupted()) {
        if ((m = (s = state) & ABITS) != WBIT) {
            if (m < RFULL) {
                if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                    return next;
            }
            else if ((next = tryIncReaderOverflow(s)) != 0L)
                return next;
        }
        if (nanos <= 0L)
            return 0L;
        if ((deadline = System.nanoTime() + nanos) == 0L)
            deadline = 1L;
        if ((next = acquireRead(true, deadline)) != INTERRUPTED)
            return next;
    }
    throw new InterruptedException();
}
```

#### readLockInterruptibly

readLockInterruptibly方法相比readLock方法增加了可中断特性。

```java
public long readLockInterruptibly() throws InterruptedException {
    long next;
    if (!Thread.interrupted() &&
        (next = acquireRead(true, 0L)) != INTERRUPTED)
        return next;
    throw new InterruptedException();
}
```

#### acquireRead

下面和acquireWrite方法一样分两部分简单分析和整理acquireRead方法的主要逻辑。

***等待线程enqueue前自旋过程***

线程在直接获取读锁失败后会进入本方法准备进入等待队列，并在正式进入队列前自旋地重试锁的获取。

线程在进入等待队列前的自旋过程大致如下：

1.如果等待队列中没有其他线程，则当前线程开始自旋地重试锁的获取。具体的自旋过程详见注释。

2.如果线程完成了一定自旋后还是没有获取到锁则准备进入等待队列。而在这个过程中线程也会在满足特定条件的情况下再次尝试获取锁的操作，直到最后真正进入队列。如果线程入列时处于COWAIT队列头部，还会经历下面要说明的***enqueue后队列头自旋***过程，否则就会进入阻塞状态了。具体的入列过程详见注释。

***cowait队列：***依附在等待队列读节点下的读节点列表。  

```java
for (int spins = -1;;) {
    WNode h;
  // 只有当等待队列中没有其他线程时才需要自旋，这时获取到锁的几率很大。
  if ((h = whead) == (p = wtail)) {
        for (long m, s, ns;;) {
            // 如果当前没有持有写锁线程则尝试获取读锁
            if ((m = (s = state) & ABITS) < RFULL ?
                U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L))
                return ns;
            // 如果当前有线程持有写锁则开始有次数限定的自旋
            else if (m >= WBIT) {
                if (spins > 0) {
                    if (LockSupport.nextSecondarySeed() >= 0)
                        --spins;
                }
                else {
                    // 在完成一个自旋周期后队列发现队列中有其他等待线程，则可以退出自旋准备入列
                    if (spins == 0) {
                        WNode nh = whead, np = wtail;
                        if ((nh == h && np == p) || (h = nh) != (p = np))
                            break;
                    }
                    spins = SPINS;
                }
            }
        }
    }
    // 如果等待队列没有初始化则尝试初始化。
    if (p == null) { // initialize queue
        WNode hd = new WNode(WMODE, null);
        if (U.compareAndSwapObject(this, WHEAD, null, hd))
            wtail = hd;
    }
    else if (node == null)
        node = new WNode(RMODE, p);
    // 读线程入列的方式和写线程稍有不同，如果队列尾部已经存在等待读的线程，则当前线程会被置于等待读线程的cowait队列中而不是设置为next节点。
    // 
    else if (h == p || p.mode != RMODE) {
        if (node.prev != p)
            node.prev = p;
        else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
            p.next = node;
            break;
        }
    }
    else if (!U.compareAndSwapObject(p, WCOWAIT,
                                     node.cowait = p.cowait, node))
        node.cowait = null;
    else {
        for (;;) {
            WNode pp, c; Thread w;
            // 释放cowait队列中的节点，可能已经获取到锁但节点没有移除
            if ((h = whead) != null && (c = h.cowait) != null &&
                U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                (w = c.thread) != null) // help release
                U.unpark(w); // 如果节点阻塞还需要唤醒
            // 如果发现等待队列空了则继续尝试获取锁
            if (h == (pp = p.prev) || h == p || pp == null) {
                long m, s, ns;
                do {
                    if ((m = (s = state) & ABITS) < RFULL ?
                        U.compareAndSwapLong(this, STATE, s,
                                             ns = s + RUNIT) :
                        (m < WBIT &&
                         (ns = tryIncReaderOverflow(s)) != 0L))
                        return ns;
                } while (m < WBIT);
            }
            // 非cowait队列头结点的线程将进入阻塞
            if (whead == h && p.prev == pp) {
                long time;
                if (pp == null || h == p || p.status > 0) {
                    node = null; // throw away
                    break;
                }
                if (deadline == 0L)
                    time = 0L;
                else if ((time = deadline - System.nanoTime()) <= 0L)
                    return cancelWaiter(node, p, false);
                Thread wt = Thread.currentThread();
                U.putObject(wt, PARKBLOCKER, this);
                node.thread = wt;
                if ((h != pp || (state & ABITS) == WBIT) &&
                    whead == h && p.prev == pp)
                    U.park(false, time);
                node.thread = null;
                U.putObject(wt, PARKBLOCKER, null);
                if (interruptible && Thread.interrupted())
                    return cancelWaiter(node, p, true);
            }
        }
    }
}
```

***enqueue后队列头自旋***

读锁获取线程如果在入列时位于cowait队列头则会自旋地重试读锁获取。如果最终失败则进入阻塞等待。

```java
// COWAIT队列头结点将在真正阻塞前自旋地重试获取锁
for (int spins = -1;;) {
    WNode h, np, pp; int ps;
    if ((h = whead) == p) {
        if (spins < 0)
            spins = HEAD_SPINS;
        else if (spins < MAX_HEAD_SPINS)
            spins <<= 1;
        for (int k = spins;;) { // spin at head
            long m, s, ns;
            // 如果自旋过程中获取锁成功则唤醒当前cowait对列中的其他读等待线程
            if ((m = (s = state) & ABITS) < RFULL ?
                U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                WNode c; Thread w;
                whead = node;
                node.prev = null;
                while ((c = node.cowait) != null) {
                    if (U.compareAndSwapObject(node, WCOWAIT,
                                               c, c.cowait) &&
                        (w = c.thread) != null)
                        U.unpark(w);
                }
                return ns;
            }
            else if (m >= WBIT &&
                     LockSupport.nextSecondarySeed() >= 0 && --k <= 0)
                break;
        }
    }
    else if (h != null) {
        WNode c; Thread w;
        while ((c = h.cowait) != null) {
            if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                (w = c.thread) != null)
                U.unpark(w);
        }
    }
    // 只要队列状态有变化，节点都不会进入阻塞而是返回循环重试
    if (whead == h) {
        // 前驱节点发生变化
        if ((np = node.prev) != p) {
            if (np != null)
                (p = np).next = node;   // stale
        }
        else if ((ps = p.status) == 0)
            U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
        // 前驱节点取消
        else if (ps == CANCELLED) {
            if ((pp = p.prev) != null) {
                node.prev = pp;
                pp.next = node;
            }
        }
        // 队列状态没有发生变化则进入阻塞
        else {
            long time;
            if (deadline == 0L)
                time = 0L;
            else if ((time = deadline - System.nanoTime()) <= 0L)
                return cancelWaiter(node, node, false);
            Thread wt = Thread.currentThread();
            U.putObject(wt, PARKBLOCKER, this);
            node.thread = wt;
            if (p.status < 0 &&
                (p != h || (state & ABITS) == WBIT) &&
                whead == h && node.prev == p)
                U.park(false, time);
            node.thread = null;
            U.putObject(wt, PARKBLOCKER, null);
            if (interruptible && Thread.interrupted())
                return cancelWaiter(node, node, true);
        }
    }
}
```
### 乐观读

StampedLock类还支持乐观读，所谓乐观读就是并不真正获取共享读锁，而是假定在发起乐观读到读取过程结束这一过程中没有线程执行写操作，如果没有就代表读操作有效，这在写操作较少的情况下可以有更好的性能。

乐观读主要有两个方法来实现，尝试乐观读的tryOptimisticRead方法和检查乐观读有效性的validate方法。

```java
// 乐观读会在有线程持有写锁时失败，如果成功则返回当前的state值用于读取结束后的有效性检查
public long tryOptimisticRead() {
    long s;
    return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
}
```

```java
// 有效性检查方法会检查读取过程中state值是否发生变化
public boolean validate(long stamp) {
    U.loadFence();
    return (stamp & SBITS) == (state & SBITS);
}
```

### 锁转换

StampedLock类还提供了可以将以上各种锁相互转换的方法，具体的方法定义如下：

#### tryConvertToWriteLock

用于根据指定stamp执行写锁转换(stamp合法的前提下)的方法，遵循下面的转换规则：

* 如果指定stamp代表持有一个写锁，则直接返回。
* 如果指定stamp代表持有一个读锁并且没有其他线程是有读锁/写锁时，释放读锁并获取写锁。
* 如果指定stamp代表乐观读，则在当前写锁可用的情况下获取。
* 以上情况以外都无法完成写锁转换

```java
public long tryConvertToWriteLock(long stamp) {
    long a = stamp & ABITS, m, s, next;
    // 首先需要检查stamp释放合法
    while (((s = state) & SBITS) == (stamp & SBITS)) {
        // 如果当前是乐观读则直接尝试获取写锁
        if ((m = s & ABITS) == 0L) {
            if (a != 0L)
                break;
            if (U.compareAndSwapLong(this, STATE, s, next = s + WBIT))
                return next;
        }
        // 当前已经持有写锁则直接返回
        else if (m == WBIT) {
            if (a != m)
                break;
            return stamp;
        }
        // 仅有当前线程持有读锁则释放读锁并获取写锁
        else if (m == RUNIT && a != 0L) {
            if (U.compareAndSwapLong(this, STATE, s,
                                     next = s - RUNIT + WBIT))
                return next;
        }
        else
            break;
    }
    return 0L;
}
```

#### tryConvertToReadLock

用于根据指定stamp执行读锁转换(stamp合法的前提下)的方法，遵循下面的转换规则：

* 如果指定stamp代表持有一个写锁，释放写锁并获取读锁。
* 如果指定stamp代表持有一个读锁则直接返回。
* 如果指定stamp代表乐观读，则在当前读锁可用的情况下获取。
* 以上情况以外都无法完成读锁转换。

```java
public long tryConvertToReadLock(long stamp) {
    long a = stamp & ABITS, m, s, next; WNode h;
    // 首先需要检查stamp释放合法
    while (((s = state) & SBITS) == (stamp & SBITS)) {
        // 当前未乐观读，则尝试获取读锁
        if ((m = s & ABITS) == 0L) {
            if (a != 0L)
                break;
            else if (m < RFULL) {
                if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                    return next;
            }
            else if ((next = tryIncReaderOverflow(s)) != 0L)
                return next;
        }
        // 当前持有写锁则释放写锁并获取读锁
        else if (m == WBIT) {
            if (a != m)
                break;
            state = next = s + (WBIT + RUNIT);
            // 释放了写锁意味着还需要唤醒等待队列中的节点
            if ((h = whead) != null && h.status != 0)
                release(h);
            return next;
        }
        // 当前持有读锁则直接返回。
        else if (a != 0L && a < WBIT)
            return stamp;
        else
            break;
    }
    return 0L;
}
```

#### tryConvertToOptimisticRead

用于根据指定stamp执行乐观读转换（stamp合法的前提下）的方法，遵循以下转换规则：

* 如果指定stamp代表持有写/读锁，则释放锁并返回一个乐观读用的stamp。

* 如果指定stamp代表乐观读，如果乐观读有效则返回stamp。

* 以上情况以外都无法完成乐观读转换。

  ```java
  public long tryConvertToOptimisticRead(long stamp) {
      long a = stamp & ABITS, m, s, next; WNode h;
      U.loadFence();
      for (;;) {
          // 如果stamp不合法则返回0
          if (((s = state) & SBITS) != (stamp & SBITS))
              break;
          // 如果当前未乐观读则在乐观读有效的情况下返回stamp
          if ((m = s & ABITS) == 0L) {
              if (a != 0L)
                  break;
              return s;
          }
          // 如果当前持有写锁则释放写锁并返回一个乐观读用的stamp
          else if (m == WBIT) {
              if (a != m)
                  break;
              state = next = (s += WBIT) == 0L ? ORIGIN : s;
              if ((h = whead) != null && h.status != 0)
                  release(h);
              return next;
          }
          // 当前state发生变化则返回0
          else if (a == 0L || a >= WBIT)
              break;
          // 当前持有读锁则释放读锁并返回一个乐观读用的stamp
          else if (m < RFULL) {
              if (U.compareAndSwapLong(this, STATE, s, next = s - RUNIT)) {
                  if (m == RUNIT && (h = whead) != null && h.status != 0)
                      release(h);
                  return next & SBITS;
              }
          }
          else if ((next = tryDecReaderOverflow(s)) != 0L)
              return next & SBITS;
      }
      return 0L;
  }
  ```