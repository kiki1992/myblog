---
title: Java并发编程--AQS应用之Semaphore
date: 2018-02-02 14:28:32
summary: 本篇对Java并发框架中应用AQS实现计数信号量的Semaphore源码进行了简单整理和记录。
tags: [多线程]
---

## 概要

> * Semaphore类通过许可数量管理的方式实现了计数信号量的功能，通常我们可以用它来限制访问某个资源的并发线程数。
> * Semaphore的许可数量可以设为1，这样做可以实现一个特殊的锁，即执行释放操作的线程可以是当前并未持有锁的其他线程，这在需要死锁恢复时很有用处。
> * Semaphore可以指定许可的获取策略是否公平，通常情况下会指定公平的获取策略来保证访问某个资源时不会有线程饿死的情况，但如果相比这一点更关心系统吞吐量的情况下可以选择非公平策略。需要注意的是，不带超时参数的tryAcquire方法即使在公平策略下也采用的是非公平的获取方式。
> * Semaphore支持指定获取/释放许可的数量，即每次可以获取/释放复数个信号量，不过这样做需要小心非公平策略下可能导致某些线程无限等待的情况。

## 主要原理

### 应用AQS实现的同步控制基础类Sync

Sync类利用AQS的state域变量来保存可用的许可数量，state等于0代表许可已经用完。

以下是Sync类中实现的获取和释放许可数量的方法：

```java
// 获取许可的方法
final int nonfairTryAcquireShared(int acquires) {
    for (;;) { // 即使还有许可量剩余，但可能发生竞争的情况，所以需要在获取失败后重试
        int available = getState();
        int remaining = available - acquires;
        // 如果许可量用完或者获取许可量失败则返回
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}

// 释放许可量的方法
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}

// 批量减少许可量的方法，通常可以在资源可用性变差时限制访问的并发数
final void reducePermits(int reductions) {
    for (;;) {
        int current = getState();
        int next = current - reductions;
        if (next > current) // underflow
            throw new Error("Permit count underflow");
        // 和acquire的区别就是即使当前已经没有许可量也不会导致阻塞
        if (compareAndSetState(current, next))
            return;
    }
}
```

### 基于Sync实现的公平锁/非公平锁

#### 公平锁FairSync

FairSync是基于Sync类实现的公平获取策略类，其释放许可的方法沿用了Sync中的实现，而获取许可的方法则使用了新的实现来保证公平策略。

以下是FairSync类实现的获取许可方法：

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors()) // 同步队列中的等待线程优先
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

#### 非公平锁NonfairSync

NonfairSync是基于Sync类实现的非公平获取策略类，其获取和释放许可的方法都直接沿用了Sync中的实现。



## 总结及代码实践

Semaphore是基于AQS实现的又一实用同步类，理解了AQS的实现原理之后Semaphore同样非常简单。

最后，我们来看一个使用Semaphore实现资源访问限制的例子。

```java
// 代码模拟了一个武器库，如果武器库里的武器都被拿完了，接下来的士兵就必须等待之前的士兵归还武器。
public class DemoArsenal {

    private int weaponCount;
    private DemoWeapon[] weapons;
    private Semaphore available;
    private boolean[] onLoan;

    public DemoArsenal(int weaponCount) {
        weapons = new DemoWeapon[weaponCount];
        for(int count = 0; count < weaponCount; count++) {
            weapons[count] = new DemoWeapon("weapon" + count);
        }
        this.weaponCount = weaponCount;
        onLoan = new boolean[weaponCount];
        available = new Semaphore(weaponCount, true);
    }

    // 归还武器的方法，调用内部维护武器库状态的同步方法
    public void returnWeapon(DemoWeapon weapon) {
        if (markAsAvailable(weapon)) {
            available.release();
        }
    }

    // 借用武器的方法，调用内部维护武器库的同步方法
    public DemoWeapon borrowWeapon() throws InterruptedException{
        available.acquire();
        return getNextAvailableWeapon();
    }

    private synchronized boolean markAsAvailable(DemoWeapon weapon) {

        for (int count = 0; count < weaponCount; count++) {
            if (weapons[count] == weapon) {
                if (onLoan[count]) {
                    onLoan[count] = false;
                    return true;
                } else {
                    return false;
                }
            }
        }

        return false;
    }

    private synchronized DemoWeapon getNextAvailableWeapon() {

        for (int count = 0; count < weaponCount; count++) {
            if (!onLoan[count]) {
                onLoan[count] = true;
                return weapons[count];
            }
        }

        return null;
    }

    public static void main(String[] args) {

        DemoArsenal demoArsenal = new DemoArsenal(3);

        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int count = 0; count < 5; count++) {
            executorService.submit(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " try to get weapon");
                    DemoWeapon weapon = demoArsenal.borrowWeapon();
                    System.out.println(Thread.currentThread().getName() + " get weapon " + weapon.getName());
                    Thread.sleep(2000);
                    System.out.println(Thread.currentThread().getName() + " return weapon " + weapon.getName());
                    demoArsenal.returnWeapon(weapon);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        executorService.shutdown();

    }

}
```