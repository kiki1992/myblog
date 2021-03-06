---
title: CountDownLatch源码笔记
date: 2017-12-14 13:11:32
summary: 本篇对CountDownLatch源码做了简单分析及整理
tags: 多线程
---

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

## 编码实例
这里针对上面提到的两个应用场景给出了简单的代码实例。
#### 一个简单的开关

代码比较简单，直接看注释就可以了。

```java
public class CountDownLatchDemo1 {

    private static final int waitThreadsCount = 5;

    public static void main(String[] args) {

        // 创建一个计数器
        CountDownLatch countDownLatch = new CountDownLatch(1);

        // 创建等待闸门开放的线程
        for (int count = 0; count < waitThreadsCount; count++) {

            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " waiting...");
                try {
                    countDownLatch.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    return;
                }
                System.out.println(Thread.currentThread().getName() + " passed through!");
            } ).start();

        }

        // 主线程睡眠2s后打开闸门
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 主线程打开闸门
        System.out.println("############################");
        System.out.println(Thread.currentThread().getName() + " open the gate!");
        System.out.println("############################");
        countDownLatch.countDown();

    }
}
```

下面是代码运行的结果：
![CountDownLatchDemo1运行结果](/myblog/images/CountDownLatchDemo1Result.png)
可以看到等待闸门打开的线程的确在执行countDown处理后恢复运行了。
***
#### 一个等待其他相关任务全部完成才能执行的任务

这里模拟了一个关机的过程，主程序必须等待关闭应用和清理工作完成后才能执行最后的关机任务。

首先定义一个相关任务基类。子类只需要实现doWork方法即可。

```java
public abstract class BaseTaskWorker implements Runnable{

    private String workName;
    private boolean finished;
    private CountDownLatch countDownLatch;

    public BaseTaskWorker(String workName, CountDownLatch countDownLatch) {
        this.workName = workName;
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {

        try {
            doWork();
        } finally {
            if (countDownLatch != null) {
                countDownLatch.countDown();
            }
        }
    }

    public String getWorkName() {
            return this.workName;
    }

    public abstract void doWork();

}
```

下面是两个具体的相关任务实现类。

首先是模拟应用关闭任务的实现类，这里让线程睡眠2s来模拟关闭应用的过程。

```java
public class CloseAppTaskWorker extends BaseTaskWorker{

    public CloseAppTaskWorker(CountDownLatch countDownLatch) {
        super("CloseAppTaskWorker", countDownLatch);
    }

    @Override
    public void doWork() {
        System.out.println(getWorkName() + " start!");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(getWorkName() + " normal end!");
    }
}
```

然后是模拟清理任务的实现类，这里同样通过线程睡眠的方式模拟清理的过程。

```java
public class CleanUpTaskWorker extends BaseTaskWorker{

    public CleanUpTaskWorker(CountDownLatch countDownLatch) {
        super("CleanUpTaskWorker", countDownLatch);
    }

    @Override
    public void doWork() {
        System.out.println(getWorkName() + " start!");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(getWorkName() + " normal end!");
    }
}
```

最后是模拟关机过程的主线程。主线程会在所有相关任务都完成后执行关机操作。

```java
public class CountDownLatchDemo2 {


    private static final int taskWorkerCount = 2;

    public static void main(String[] args) {

        CountDownLatch countDownLatch = new CountDownLatch(taskWorkerCount);

        List<BaseTaskWorker>  taskWorkerList = new ArrayList(taskWorkerCount);
        taskWorkerList.add(new CloseAppTaskWorker(countDownLatch));
        taskWorkerList.add(new CleanUpTaskWorker(countDownLatch));

        for (BaseTaskWorker taskWorker : taskWorkerList) {
            new Thread(taskWorker).start();
        }

        System.out.println("waiting for task workers...");
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("ready for shutdown!");
    }
}
```

下面是代码运行的结果：
![CountDownLatchDemo1运行结果](/myblog/images/CountDownLatchDemo2Result.png)
可以看到主线程确实是在应用关闭和清理任务都完成之后才恢复运行的。