# CountDownLatch源码笔记
## 主要特性
> * 1.CountDownLatch是并发编程的一个辅助类，允许一个或多个线程等待一组特定线程完成某项任务后再执行。
> * 2.CountDownLatch在初始化时需要制定一个初始计数值。调用await方法的线程会在这个数值达到0之前阻塞，当特定线程调用同一个计数器的countDown方法后，计数值后递减。一旦计数值到达0，所有调用await方法导致阻塞的线程将立即恢复运行。需要注意的是，初始设置的计数值一旦设置就无法修改，如果需要可以修改的计数器版本，可以考虑使用CyclicBarrier。
> * 3.你可以使用CountDownLatch实现诸如以下这样的需求：
>   (1 一个简单的开关：允许在一个线程打开开关前让其他所有调用await方法的线程阻塞直到该线程调用countDown方法打开开关。要实现这个需求只需要初始化一个计数值为1的计数器。
>   (2 一个等待其他相关任务全部完成才能执行的任务：允许某个调用await方法的线程等待执行相关任务的其他线程调用countDown方法直至计数值为0。要实现这个功能则需要初始化一个计数值等于相关任务数的计数器。

## 主要变量
CountDownLatch内部仅维护以下一个变量，所有CountDownLatch类方法都是基于这个变量实现的。

```java
private final Sync sync;
```

而Sync又是一个继承自AbstractQueuedSynchronizer的内部类，以下是这个类的实现代码。

其中tryAcquireShared和tryReleaseShared都是对父类方法的重写，并且这两个方法都是由父类中定义的其他方法调用的。

关于AbstractQueuedSynchronizer的源码实现会在另一篇博客中做具体的记录。

```java
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    // 在await调用的AbstractQueuedSynchronizer类方法中使用，判断是否到达await结束状态
    // getState返回0说明已经到达await结束状态
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
  
    // 在countDown调用的AbstractQueuedSynchronizer类方法中使用，将计数值减1并在计数值到达0时     // 发出信号(返回true)
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

## 构造器

CountDownLatch类仅提供一个以计数值为入参的构造器方法。

该构造方法使用入参指定的计数值初始化用于await和countDown方法调用的Sync实例。

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

## 主要方法

CountDownLatch类主要有三个方法，一个countDown方法，两个await方法(是否带超时属性)。

需要注意的一点是await方法可以抛出InterruptedException，说明其可以被Thread的interrupt方法打断。

所有这些方法的实现都调用了AbstractQueuedSynchronizer类中定义的方法，有关这些方法会在另一篇记录AbstractQueuedSynchronizer类源码的博客中予以说明。

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
public boolean await(long timeout, TimeUnit unit)
	throws InterruptedException {
	return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
public void countDown() {
	sync.releaseShared(1);
}
```