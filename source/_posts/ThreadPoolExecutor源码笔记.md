---
title: ThreadPoolExecutor源码笔记
date: 2018-01-03 15:28:32
summary: 本篇对ThreadPoolExecutor源码做了简单分析及整理。
tags: 多线程
---

## ThreadPoolExecutor源码

```java
public class ThreadPoolExecutor extends AbstractExecutorService
```

### 概要

> * ThreadPoolExecutor使用线程池中的可用线程来执行提交的任务，通常可以使用Executors类的工厂方法来获取不同配置的ThreadPoolExecutor实例。
>
> * 使用线程池的方法解决了两个问题：其一在执行大量异步任务时改善了程序性能，其二是提供了资源管理及限定的方法。ThreadPoolExecutor同时还维护了诸如执行完成线程数这样的统计信息。
>
> * 尽管ThreadPoolExecutor提供了许多可供调整配置的参数以及可扩展Hook，Oracle官方还是极力推荐使用Executors的工厂方法来创建常用的ThreadPoolExecutor实例。具体的例子如下：
>
>   ***Executors.newCachedThreadPool()*** -- 用于创建无界线程池，具备自动扩容功能
>
>   ***Executors.newFixedThreadPool(int)*** -- 用于创建拥有指定线程数的线程池
>
>   ***Executors.newSingleThreadExecutor()*** -- 用于创建单线程线程池 
>
>   * 如果不使用工厂方法来创建ThreadPoolExecutor实例，则需要在配置调整本类时遵循以下规则：
>
>     ***核心线程数及最大线程数***
>
>     ThreadPoolExecutor会根据核心线程数以及最大线程数的设值自动调整线程池大小。当有新任务被提交时，如果当前运行的线程数少于核心线程数，则新提交的任务会由新创建的线程执行，即使此时还有空闲线程。而当运行线程大于核心线程数且小于最大线程数时，只有当队列满时才会创建新线程。因此如果将核心线程数和最大线程数设置为相同值即构建了一个固定线程数的线程池，而将最大线程数设置为Integer最大值意味着可以接受任意数量的并发任务。
>
>     核心线程数和最大线程数通常仅在构造方法中指定，如果需要也可以通过setCorePoolSize(int) 和setMaximumPoolSize(int)动态设置。
>
>     ***按需构造***
>
>     默认情况下，即使是核心线程也只会在有任务实际提交时被创建并启动。可以通过prestartCoreThread()或prestartAllCoreThreads()方法提前启动核心线程。
>
>     ***创建新线程***
>
>     ThreadPoolExecutor使用ThreadFactory来创建新线程，如果没有指定ThreadFactory则默认使用Executors.defaultThreadFactory()。默认创建的线程属于同一线程组，优先级相同(NORM_PRIORITY) 且都是非守护线程。可以指定其他ThreadFactory来达到修改线程名，所在组，优先级等属性。如果ThreadFactory创建新线程失败，executor将会继续，但可能无法执行任何任务。线程需要拥有modifyThread运行时权限，如果工作者线程或其他使用线程池的线程没有此权限，则服务可能退化：配置的修改可能无法及时生效，关闭的线程池也可能处于可以终止但无法完成终止的状态。
>
>     ***Keep-alive时间***
>
>     如果线程池中线程数超过核心池大小，则多出的线程将在保持空闲时间达到keepAliveTime 设值后被销毁，这样做可以在线程池非活跃时减少资源消耗。如果在这之后线程池重新变得活跃，则新线程将被创建。keep-alive时间可以通过setKeepAliveTime(long, TimeUnit)方法动态设置，将时间设置为Long.MAX_VALUE TimeUnit.NANOSECONDS 将在线程池关闭之前禁止空闲线程的销毁。
>
>     默认情况下，只有当线程数超过核心池大小才会适用keep-alive策略。如果需要对核心线程也适用keep-alive策略，可以通过allowCoreThreadTimeOut(boolean)设置，前提是keep-alive时间设值不为0。
>
>     ***队列***
>
>     任何BlockingQueue都可以用于传递和保持提交的任务。队列的使用和线程池的大小相互作用关系如下：
>
>     1.如果线程数小于核心池大小，则Executor会选择新建线程而不是将任务放入队列。
>
>     2.如果线程数大于等于核心池大小，则Executor会选择将任务放入队列。
>
>     3.如果无法将任务放入队列且当前线程数小于最大池大小时将会创建新线程，否则任务将被拒绝。
>
>     队列有以下三个一般策略：
>
>     1.***直接传递：*** 使用SynchronousQueue作为工作队列将把提交的任务直接传递给线程。因此如果当前无空闲线程可用则将任务放入队列的操作会失败，新线程将被创建。这种策略可以避免在持有一组可能存在内部依赖关系的任务时发生锁定。这种策略通常需要设置最大池大小无上限来避免任务被拒绝，这也意味着如果任务的执行速度跟不上提交速度线程数可能无限增长。
>
>     2.***无边界队列：*** 使用无界队列(比如未定义容量的LinkedBlockingQueue)作为工作队列将会导致所有核心线程繁忙时，新提交的任务被放入队列中等待。因此不会有超过核心池大小的线程被创建，这种情况下最大池大小将不起作用。当所有任务完全独立时采用这种策略是合适的选择，比如网页服务器。尽管这种策略可以平滑瞬时请求压力，但同时也意味着当任务的执行速度跟不上提交速度时工作队列可能无限增长。
>
>     3.***有边界队列：*** 使用有界队列有助于防止资源过度消耗，但同时也更难协调和操控。队列长度和线程池最大容量可能会互相影响，此消彼长。使用大队列小线程池可以减少CPU开销，OS资源使用以及上下文切换带来的间接开销，却可能导致较差的吞吐量。而使用较小的队列通常需要较大容量的线程池，这将使CPU保持繁忙却可能导致无法接受的调度开销，从而使得系统吞吐量变差。
>
>     ***拒绝任务***
>
>     当新任务被提交时，如果遇上Executor关闭，或者Executor处于饱和状态时，程序将会调用RejectedExecutionHandler.rejectedExecution(Runnable, ThreadPoolExecutor)方法来拒绝任务的执行。官方默认提供了以下四种拒绝策略：
>
>     1.***ThreadPoolExecutor.AbortPolicy*** 默认策略会抛出RejectedExecutionException异常。
>
>     2.***ThreadPoolExecutor.CallerRunsPolicy*** 此策略会在使用execute方法的线程中执行任务，即直接调用Runnable对象的run方法，这有助于放缓任务提交的速度。
>
>     3.***ThreadPoolExecutor.DiscardPolicy*** 此策略将静默丢弃任务。
>
>     4.***ThreadPoolExecutor.DiscardOldestPolicy*** 此策略将丢弃队列中最初的任务然后重试，如果任务提交依旧失败则会继续丢弃旧任务。
>
>     如果需要可以实现RejectedExecutionHandler接口来自定义拒绝策略。
>
>     ***Hook方法***
>
>     本类提供了protected方法beforeExecute(Thread, Runnable)和afterExecute(Runnable, Throwable) ，分别在任务执行前后调用。可以利用这些方法来实现对执行环境的操控：比如ThreadLocals的重新初始化，数据收集，日志记录等。
>
>     另外还可以重写terminated方法来触发Executor完全终止后的一些特殊处理。
>
>     如果hook或者回调方法抛出异常，则相关工作者线程将会依次失败并意外终止。
>
>     ***队列维护***
>
>     可以利用getQueue方法访问队列来达到监控及debug目的。	除此之外不应该对队列做其他操作。
>
>     另外当队列中的大量任务被取消时，可以利用remove(Runnable r)和purge方法来协助空间回收。其中remove方法可以从队列中移除指定Runnable对象，但如果Runnable对象在被放入队列前经过包装将无法使用此方法移除，这种情况下可以使用purge方法，此方法可以将队列中已经取消的Future对象移除。
>
>     ***结束操作***
>
>     如果线程池在程序中不再被引用并且不存在剩余线程时将会被自动关闭。如果想要确保即使在用户忘记调用shutdown方法时线程池也能在没有引用后被回收，就需要设置恰当的keep-alive时间来确保空闲线程会最终死亡，并将核心线程数设置为0或者设置allowCoreThreadTimeOut(boolean)。

​	

### 重要变量及内部类

#### 内部类

**任务拒绝策略内部实现类**

所有拒绝策略类都实现了RejectedExecutionHandler接口，要实现自定义的拒绝策略只需实现同一接口即可。以下是默认定义的拒绝策略实现类：

***CallerRunsPolicy -- 使用调用者线程执行任务的策略***

```java
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        
        public CallerRunsPolicy() { }

        // 使用调用者线程执行任务，如果Executor关闭则任务被丢弃
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```

***AbortPolicy -- 异常抛出策略***

```java
    public static class AbortPolicy implements RejectedExecutionHandler {
      
        public AbortPolicy() { }

        // 直接抛出RejectedExecutionException
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```

***DiscardPolicy -- 静默丢弃策略***

```java
    public static class DiscardPolicy implements RejectedExecutionHandler {
        
        public DiscardPolicy() { }

        // 不做任何处理，直接将任务静默丢弃
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```

***DiscardOldestPolicy -- 舍弃最旧任务策略***

```java
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        
        public DiscardOldestPolicy() { }

        // 丢弃队列中最旧的任务并尝试重新执行任务，如果executor已经关闭则将任务丢弃
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

**Worker类**

Worker类的主要工作是为执行任务的线程维护中断控制状态，并提供一些简单的记录功能，例如执行完成的任务数。

本类继承自AbstractQueuedSynchronizer类，利用其来简化执行各个任务时获取/释放锁的实现过程。具体是实现了一个不可重入的锁来替代ReentrantLock的使用，这样一来在其调用诸如setCorePoolSize的线程池控制方法时就无法重新获取锁了。

另外，在本类中还实现了在线程真正执行任务前抑制中断操作的处理。

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    // 如果worker尚未启动，interrupt处理将被抑制
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

#### 重要变量

***completedTaskCount***

completedTaskCount变量用于记录任务完成数，只有当worker终止时才会更新变量值，访问此变量只能是在加锁的情况下。

```java
private long completedTaskCount;
```

***Executor容量&运行状态变量***

ThreadPoolExecutor使用Integer的低29位来存储Worker数量，而高位则被用来存储Executor的运行状态。相关变量如下：

```java
// 用于控制记录utor状态及worker线程数
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 用于存储Worker数量的位数
private static final int COUNT_BITS = Integer.SIZE - 3; 
// 得到低位皆为1的结果，代表Executor的最大容量
private static final int CAPACITY   = (1 << COUNT_BITS) - 1; 

// 高位用于存储Executor的运行状态，各个状态代表的意义和转换过程如下：
// RUNNING -- 接受新任务并执行队列中的任务
// SHUTDOWN -- 不再接受新任务但继续执行队列中的任务
// STOP -- 不再接受新任务也不会继续执行队列中的任务，同时还会中断正在执行的任务
// TIDYING -- 所有任务已经终止，Worker线程已经不存在，处于该状态的线程将执行terminated() Hook方法
// TERMINATED -- terminated() Hook方法执行完成
// ##状态转换过程##
// RUNNING->SHUTDOWN:调用shutdown方法
// RUNNING/SHUTDOWN->STOP:调用shutdownNow方法
// SHUTDOWN->TIDYING:队列中任务执行完毕且所有Worker线程已经销毁
// STOP->TIDYING:所有Worker线程已经销毁
// TIDYING->TERMINATED:terminated() Hook方法完成执行
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

***

### 主要方法

#### Executor/ExecutorService方法

***execute***

execute方法将使指定任务在未来某个时刻执行，如果任务无法被提交将由RejectedExecutionHandler作适当处理。

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
  
    int c = ctl.get();
    // 首先，如果当前worker线程数小于核心线程数，则直接新增worker线程来执行指定任务。
    // 在addWorker方法里会检查Executor，线程池状态，因此这里无需额外检查
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 如果任务不能被直接执行，则尝试将其放入队列中
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 将任务放入队列后如果executor状态发生变化，则任务将可能从队列中移除并拒绝。
        // 另外，还需要检查是否需要新增worker线程来处理任务。
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 如果将任务放入队列的操作失败，则需要尝试新增worker线程来直接处理任务
    // 如果到这一步新增Worker线程失败，说明executor已经关闭或者饱和，当前任务将被拒绝。
    else if (!addWorker(command, false))
        reject(command);
}
```

***shutdown***

shutdown方法将发起一个有序的关闭过程，已经提交的任务将被执行，而新任务的提交将被拒绝。

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN); 
        interruptIdleWorkers(); // 给空闲线程发送中断信号
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

***shutdownNow***

shutdownNow方法将试图停止正在运行的任务，阻止等待中任务的执行，将这些任务从等待队列中移除并返回给调用者。本方法的其他特性和父类方法一致。

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers(); // 向所有启动的工作者线程发送中断信号
        tasks = drainQueue(); //drainQueue方法将队列中的任务移除并添加到返回列表中
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}

    private List<Runnable> drainQueue() {
        BlockingQueue<Runnable> q = workQueue;
        ArrayList<Runnable> taskList = new ArrayList<Runnable>();
        // 使用queue的drainTo方法将队列元素移动至指定集合
        q.drainTo(taskList);
        // 调用某些种类Queue的drainTo方法可能不能完全移除所有元素
        // 这种情况需要逐个移除未被移除的元素
        if (!q.isEmpty()) {
            for (Runnable r : q.toArray(new Runnable[0])) {
                if (q.remove(r))
                    taskList.add(r);
            }
        }
        return taskList;
    }
```
#### Hook方法

ThreadPoolExecutor提供了一系列可供子类重写的Hook方法，来实现对任务执行以及Executor关闭的监控管理等操作。

***beforeExecute***

```java
protected void beforeExecute(Thread t, Runnable r) { }
```

beforeExecute方法在任务执行前由执行任务的线程调用，为了能适当地处理多重嵌套重写，子类一般需要在重写方法的最后调用super.beforeExecute

***afterExecute***

```java
protected void afterExecute(Runnable r, Throwable t) { }
```

afterExecute方法在任务执行完成后由执行任务的线程调用，当任务由于RuntimeException或是Error导致意外终止时，入参Throwable会被设置。为了能适当地处理多重嵌套重写，子类一般需要在重写方法的开头调用super.afterExecute。

**需要注意的是：** 当执行动作被封闭在任务中时（比如FurtureTask），将由任务内部处理执行过程中的异常，因此任务不会因此而意外终止，而这些异常也将不会被传递至本方法。针对这种情况可以做一些特殊处理，比如下面这个例子：

```java
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);

        // 如果Throwable对象没有设置，检查任务是否被包装
        if (t == null && r instanceof Future) {

            Future future = (Future) r;
            try {
                future.get();
            } catch (CancellationException ce) {
                t = ce;
            } catch (ExecutionException ee) {
                t = ee.getCause(); // 如果是ExecutionException，则实际异常被包装
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
            }
        }

        if (t != null) {
            // 如果Throwable对象存在，做某些处理
        }

    }
```

***terminated***

```java
protected void terminated() { }
```

terminated方法在Executor终止时被调用，为了能适当地处理多重嵌套重写，子类一般需要在重写方法中调用super.terminated。



#### 线程池相关方法

***prestartAllCoreThreads/prestartCoreThread/ensurePrestart***

采用默认策略时即便是核心线程也只有在真正有任务需要执行时才会被创建，调用本方法将重写这种默认策略，使核心线程提前启动，等待任务提交。

另外还有prestartCoreThread和ensurePrestart方法用于启动单个核心线程。

```java
public int prestartAllCoreThreads() {
    int n = 0;
    while (addWorker(null, true))
        ++n;
    return n;
}
void ensurePrestart() {
  int wc = workerCountOf(ctl.get());
  // 确保执行有一个线程启动
  if (wc < corePoolSize)
    addWorker(null, true);
  else if (wc == 0)
    addWorker(null, false);
}
public boolean prestartCoreThread() {
  return workerCountOf(ctl.get()) < corePoolSize &&
    addWorker(null, true);
}
```

***remove***

remove方法可以从队列中移除指定Runnable对象，但如果Runnable对象在被放入队列前经过包装将无法使用此方法移除，这种情况下可以使用purge方法。

```java
public boolean remove(Runnable task) {
    boolean removed = workQueue.remove(task);
    tryTerminate(); // In case SHUTDOWN and now empty
    return removed;
}
```

***purge***

purge方法尝试从队列中移除所有已经取消的Future任务。本方法可能因为执行过程中受到其他线程干扰而失败。当发生干扰后，本方法会采用一种较慢的方式来继续执行移除处理。

```java
public void purge() {
    final BlockingQueue<Runnable> q = workQueue;
    try {
        // 首先尝试用迭代器移除取消任务
        Iterator<Runnable> it = q.iterator();
        while (it.hasNext()) {
            Runnable r = it.next();
            if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                it.remove();
        }
    } catch (ConcurrentModificationException fallThrough) {
        // Take slow path if we encounter interference during traversal.
        // Make copy for traversal and call remove for cancelled entries.
        // The slow path is more likely to be O(N*N).
        // 如果迭代移除过程中受到其他线程干扰，将使用队列的拷贝遍历任务并调用
        // 队列的remove方法移除取消任务
        for (Object r : q.toArray())
            if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                q.remove(r);
    }

    tryTerminate(); // In case SHUTDOWN and now empty
}
```



#### 私有方法

**Worker相关方法**

***addWorker***

addWorker方法会检查线程池状态来决定是否可以新增Worker，如果满足新增条件则新的Worker将被创建并启动，然后开始执行指定任务。如果线程池已经停止服务或者即将关闭，线程工厂创建新线程失败，则新Worker不会被创建。方法具体实现参考以下代码注释：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 新建Worker前首先会检查Executor以及队列状态
        // 如果Executor处于关闭状态或者队列为空则不会新建Worker
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty())) // 检查队列是否为空的操作只有当需要时才会被执行
            return false;

        for (;;) {
            // 接着会检查当前Worker数量来判断新Worker是否应该被创建
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 尝试更新Worker数量，Worker数量或者Executor运行状态
            // 的改变将导致更新失败。
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 这里将检查Worker数量更新失败的原因，如果失败由Worker数量变化导致
            // 将重试更新，否则重新检查Executor状态
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    // 如果以上处理成功，则进入到新增Worker的处理
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        // 创建Worker后首先会检查对应线程是否被成功创建
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                // 获取锁后会重新检查Executor状态来确保其还在运行。
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 这里还会检查线程状态来确保其可以启动
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 如果上述检查都通过了，新增的worker将被加入全局的workers Set
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 最终worker将被启动
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 如果新增启动Worker过程失败，以上处理将被回滚
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
***addWorkerFailed***

addWorkerFailed方法用于Worker线程创建失败后的回滚操作。主要处理如下：

1.如果指定了Worker，则将其从全局的workers Set中移除

2.将worker总数减1

3.尝试终止Executor的操作，以防当前worker的存在阻碍Executor终止的情况发生。

```java
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w); // 移除当前worker
        decrementWorkerCount(); // worker总数减1
        tryTerminate(); // 尝试终止操作
    } finally {
        mainLock.unlock();
    }
}
```

***runWorker***

runWorker方法会不断从队列中取出并执行任务，其处理过程中针对以下几个问题做了相应处理：

1.***任务执行和worker销毁：***启动该方法时通常已经指定了初始任务，此时便不会从对列中获取任务。否则就通过getTask方法从对列中获取需要执行的任务。如果线程池状态改变或者其他配置发生变化导致worker需要结束则其将被正常销毁，而如果是因为内部代码抛出异常导致worker需要结束也会有相应的销毁处理。需要如何销毁将由内部变量completedAbruptly的记录值决定。

2.***任务执行干扰：***在执行任务之前程序就首先获得锁来保证除线程池停止以外，任务执行线程都不会被设置为中断状态。

3.***beforeExecute异常：***所有任务执行之前都会首先调用beforeExecute方法，如果此方法抛出异常，任务将不会被执行，worker也将被销毁。

4.***任务执行异常：***如果beforeExecute方法正常结束，任务将被最终执行。而执行过程中产生的异常会被记录并传递至afterExecute方法，RuntimeException、Error以及其他Throwables会被区分处理。任何thrown异常都将导致worker被销毁。

5.***afterExecute异常：***afterExecutte方法抛出异常也将导致worker被销毁。

以上关于异常的处理机制也保证了我们可以为aferExecute方法或者线程的UncaughtExceptionHandler提供尽可能准确的异常信息。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 如果线程池正在关闭，需要确保线程被中断，否则需要清除线程中断状态
            // 这里保证了只有线程池的关闭操作才会影响到后续任务的执行
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            // 任务执行
            try {
                // 任务前处理
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                // 上面提到的异常处理
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 任务后处理
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // worker销毁处理
        processWorkerExit(w, completedAbruptly);
    }
}
```

***getTask***

getTask方法会根据当前配置决定在获取任务时是阻塞还是超时等待。如果worker线程因为以下原因需要终止则方法将返回null。

1.当前worker线程数大于maximumPoolSize（调用setMaximumPoolSize将maximumPoolSize设为一个较小值）

2.线程池已经停止。

3.线程池关闭并且队列为空。

4.当前worker等待任务超时+超时worker在超时等待开始和结束时都满足需要被终止的条件（需要满足allowCoreThreadTimeOut || workerCount > corePoolSize）+如果队列非空当前worker不是线程池中最后一个worker线程。

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 如果Executor已经停止服务或者Executor已经关闭且任务队列为空
        // 则当前worker可以销毁
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // 如果允许核心线程超时/当前worker线程数已经超过核心线程数，则执行的是超时等待
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 满足上述1，4条件则可以销毁当前Worker线程
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

***interruptIdleWorkers/interruptWorkers***

interruptIdleWorkers方法将给那些正在等待任务的线程发送中断信号，线程在接收到信号后可以检查当前终止状态及配置的变化来调整自身行为。

这里需要注意的是入参的onlyOne，如果onlyOne为true，则方法只会像至多一个等待线程发送中断信号，随后中断信号将从此线程向其他等待线程扩散。这是因为每个worker线程在销毁是都会调用tryTerminate方法，而在tryTerminate方法又会调用到本方法。

interruptWorkers方法将给所有已经启动的工作线程发送中断信号。

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            //tryLock成功代表线程目前无任务正在执行
            if (!t.isInterrupted() && w.tryLock()) { 
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}

private void interruptWorkers() {
  final ReentrantLock mainLock = this.mainLock;
  mainLock.lock();
  try {
    for (Worker w : workers)
      w.interruptIfStarted();
  } finally {
    mainLock.unlock();
  }
}
```

***processWorkerExit***

processWorkerExit方法的主要职责是清理和记录即将销毁的worker线程，本方法的调用只会由worker线程发起。

本方法会把当前worker线程从全局的worker Set中移除，并检查线程池是否需要关闭，是否需要新增worker线程来替代当前worker线程。

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果Worker线程因为异常而需要被销毁，需要调整worker线程总数
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount(); // decrementWorkerCount方法会执行死循环直到调整成功

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    // Executor尚未关闭，判断是否需要新增worker线程
    if (runStateLessThan(c, STOP)) {
        // 如果当前worker线程因为异常退出，需要新增Worker线程
        // 否则根据线程池及队列情况判断是否需要新增
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}

private void decrementWorkerCount() {
  do {} while (! compareAndDecrementWorkerCount(ctl.get()));
}
```
**Executor生命周期相关方法**

***tryTerminate***

tryTerminate方法会在满足以下任意条件时将executor状态设置为TERMINATED：

1.executor已经关闭，并且线程池和队列都为空

2.executor已经停止并且线程池为空

如果不满足以上两种情况但需要终止executor时，将会检查是否还有worker线程存在，如果还有Worker线程，本方法会向任一空闲线程发送中断信号，并由该线程负责将中断信号扩散。

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 如果不满足executor终止条件，直接返回
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        // 如果executor需要终止，向任一worker线程发送中断信号
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 首先尝试将状态修改为TIDYING，失败则重试
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // 执行终止回调方法
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    // 向所有等待锁的线程发送信号
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```