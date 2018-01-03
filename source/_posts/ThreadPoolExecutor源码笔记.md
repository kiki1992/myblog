---
title: ThreadPoolExecutor源码笔记(更新中)
date: 2017-12-20 12:02:32
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
>
>     ​
>
>     ​

​	