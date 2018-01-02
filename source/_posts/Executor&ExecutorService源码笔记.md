---
title: Executor相关接口源码笔记
date: 2017-12-20 12:02:32
summary: 本篇对Executor及相关接口源码做了简单分析及整理。
tags: 多线程
---
## Executor接口
```java
public interface Executor 
```
### 概要
> * Executor用于执行提交的任务，而这些任务所对应的类都应该实现Runnable接口。利用Executor可以将提交任务的过程与具体执行任务的过程解耦。包括任务的具体执行机制，线程的使用细节，调度机制等等都可以依赖于Executor的具体实现。
> * Executor通常可以用来替代显示创建线程的操作，比如下面这个例子：
>
```java
  // 显示创建线程
  new Thread(new RunnableTask()).start();
  // 通过Executor隐藏线程创建细节
  Executor executor = someExecutor;
  executor.execute(new RunnableTask());
```
> * Executor接口并没有强制要求执行的过程必须是异步完成的。因此，可以像下面的例子那样直接在execute方法调用线程上立即执行提交的任务。
```java
 class DirectExecutor implements Executor {
 	public void execute(Runnable r) {
 		r.run();
  	}
 }
```
> * 当然，你还可以像下面这样为每一个任务创建一个新的线程。
```java
  class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
      new Thread(r).start();
    }
  }
```
>  *注：以上例子摘自官方API文档

### 主要方法
Executor接口只定义了一个execute方法，提交的任务会在execute方法被调用后的某个时刻被执行，而具体的执行细节会在具体的实现类里得到体现。
```java
// 如果Executor拒绝任务的提交，则会抛出RejectedExecutionException
void execute(Runnable command);
```


## ExecutorService接口
```java
public interface ExecutorService extends Executor
```
### 概要
> * ExecutorService接口继承自Executor接口，提供了管理结束以及返回Future来追踪一个或多个异步任务的方法。
> * ExecutorService可以被关闭，关闭将导致之后提交的任务被拒绝。有两种关闭ExecutorService的方法：调用shutDown方法允许正在执行的任务继续执行但会拒绝新提交的任务，而调用shutDownNow方法则会尝试中止正在执行的任务。当ExecutorService到达关闭状态以后，将不会有正在执行或等待执行的任务，新的任务也将无法被提交。一个无用或者使用完毕的ExecutorService需要被关闭以释放相关资源。

### 主要方法
**shutdown**
本方法允许正在执行的任务继续执行，同时拒绝之后新提交的任务。如果调用方法时已经处于关闭状态则不会有任何附加影响。
本方法不会等待执行中的任务完成后再返回，如果需要这样的效果可以调用awaitTermination方法。
```java
void shutdown();
```
**shutdownNow**
本方法会尝试中止所有正在执行的任务，不在处理正在等待的任务，并返回等待执行的任务列表。
需要注意的是：本方法没法保证很好地中止任务的执行。举例来说，通常本方法的实现使用***Thread.interrupt***来取消任务，如果任务无法响应interrupt操作将无法中止。
```java
List<Runnable> shutdownNow();
```
**isShutdown**
判断executor是否已经关闭。
```java
List<Runnable> shutdownNow();
```

**isTerminated**
判断执行关闭操作后是否所有任务执行完成(包括正常执行完成以及被中断的任务)。只有先调用shutdown或者shutdownNow方法，isTerminated才有可能返回true。
```java
boolean isTerminated();
```

**awaitTermination**
调用本方法会导致线程阻塞，直到以下情况发生：
* 关闭操作执行后所有任务执行完成
* 超过设定的等待时间
* 当前线程被打断

```java
boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException;
```

**submit**
如果需要在任务执行成功后获取执行结果／捕获任务执行过程中发生的异常，可以使用submit方法。
ExecutorService提供了submit的三种重载方式。
```java
// 以Callable为入参的submit方法，调用返回值Future的get方法可以获取
// 任务的执行结果
<T> Future<T> submit(Callable<T> task);
// 以Runnable和指定结果为入参的submit方法，调用返回值Future的get方法可以获取
// 指定的结果
<T> Future<T> submit(Runnable task, T result);
// 以Runnable为入参的submit方法，调用返回值Future的get方法返回null
Future<?> submit(Runnable task);
```

**invokeAll**
如果需要批量执行任务，可以使用invokeAll方法。
执行invokeAll方法会导致阻塞，直到所有任务执行完成。如果等待过程中当前线程被打断，则未完成执行的任务将会被取消。
需要注意的是调用返回的Future.isDone一定会返回true，但实际任务可能成功执行也可能是因为异常而中止。
```java
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException;
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                              long timeout, TimeUnit unit)
    throws InterruptedException;    
```

**invokeAny**
如果只要多个任务中的任意任务执行成功就能满足需求的话，可以使用invokeAny方法。
执行invokeAll方法会导致阻塞，直到任意任务执行完成(不能是异常中止)，如果最后没有任务能正常执行
完成就会导致ExecutionException。方法返回或者当前线程被打断时，未执行完的任务将会被取消。
```java
<T> T invokeAny(Collection<? extends Callable<T>> tasks)
    throws InterruptedException, ExecutionException;
<T> T invokeAny(Collection<? extends Callable<T>> tasks,
                long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
```

## ScheduledExecutorService接口
```java
public interface ScheduledExecutorService extends ExecutorService 
```
### 概要
> * ScheduledExecutorService继承自ExecutorService，定义了任务的延时/定时执行方法。
> * schedule相关方法会延时创建任务并返回一个可供取消/检查执行的ScheduledFuture对象。如果指定延时时间<=0则任务会被立即创建并执行。
> * 在schedule相关方法中，scheduleAtFixedRate和scheduleWithFixedDelay会定期创建及执行任务直到取消或者其它导致定时任务无法继续的情况发生。
> * 另外，所有schedule相关方法都以相对时间作为参数而非绝对时间或日期。其执行不依赖于系统时间，这也是和Timer类不同的地方。另外Timer不会捕获执行任务中发生的异常，因此如果任务类中抛出异常将导致同一Timer管理的任务全部无法继续执行。而ScheduledExecutorService中一个任务的异常中止不会对其它任务产生影响。

### 主要方法
**schedule**
schedule方法用于创建延时执行任务，支持Runnable/Callable实现类作为指定任务。
```java
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay, TimeUnit unit);
public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                       long delay, TimeUnit unit);                                   
```
**scheduleAtFixedRate**   
scheduleAtFixedRate用于创建按照指定时间间隔执行的任务。这里的时间间隔并非绝对，当某次任务执行时间超过时间间隔时，该任务下一次执行会等本次执行完成后开始，而不是并行执行。
如果某次任务执行过程中发生异常，该任务的后续执行计划会被抑制。
如果需要中止任务计划，可以使用ScheduledFuture取消任务或者关闭ScheduledExecutorService。
```java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay,
                                              long period,
                                              TimeUnit unit);
```
**scheduleWithFixedDelay**
scheduleWithFixedDelay用于创建按照指定延时重复执行的任务，也就是说某次任务的执行会在上一次任务执行完成并经过指定延时时间后执行。本方法的其它特性和scheduleAtFixedRate类似。
```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit);
```



## AbstractExecutorService接口
```java
public abstract class AbstractExecutorService implements ExecutorService
```
### 概要
> * 提供了对ExecutorService接口部分方法的默认实现。

### 主要方法
**newTaskFor**
将Runnable或Callable实现类对象包装成RunnableFuture接口实现类对象。返回的RunnableFuture拥有执行任务，任务取消以及执行结果获取的功能。
默认实现返FutureTask，可以通过重写newTaskFor方法返回自定义的RunnableFuture实现类对象。
```java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
}
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
}
```

**submit**
submit方法的所有默认实现都会调用newTaskFor方法对传入的Runnable或Callable实现类对象进行包装，然后调用execute执行任务。
```java
public Future<?> submit(Runnable task) {
  if (task == null) throw new NullPointerException();
  RunnableFuture<Void> ftask = newTaskFor(task, null);
  execute(ftask);
  return ftask;
}
public <T> Future<T> submit(Runnable task, T result) {
  if (task == null) throw new NullPointerException();
  RunnableFuture<T> ftask = newTaskFor(task, result);
  execute(ftask);
  return ftask;
}
public <T> Future<T> submit(Callable<T> task) {
  if (task == null) throw new NullPointerException();
  RunnableFuture<T> ftask = newTaskFor(task);
  execute(ftask);
  return ftask;
}
```
**invokeAny**
对ExecutorService的invokeAny方法的默认实现，其中调用了内部方法doInvokeAny，主要逻辑参考invokeAny。
```java
public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
	throws InterruptedException, ExecutionException {
  try {
  	return doInvokeAny(tasks, false, 0);
  } catch (TimeoutException cannotHappen) { // 非超时等待，因此不可能发生超时异常
  	assert false;
  	return null;
  }
}

public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
	long timeout, TimeUnit unit)
	throws InterruptedException, ExecutionException, TimeoutException {
	return doInvokeAny(tasks, true, unit.toNanos(timeout));
}
```
**doInvokeAny**
doInvokeAny方法内部利用ExecutorCompletionService来执行任务，并使用其poll，take方法来获取任务的执行结果。这样就可以在有任何一个任务执行完成后立即获得结果。
```java
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                              boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (tasks == null)
            throw new NullPointerException();
        int ntasks = tasks.size();
        if (ntasks == 0)
            throw new IllegalArgumentException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);

        // For efficiency, especially in executors with limited
        // parallelism, check to see if previously submitted tasks are
        // done before submitting more of them. This interleaving
        // plus the exception mechanics account for messiness of main
        // loop.

        try {
            // Record exceptions so that if we fail to obtain any
            // result, we can throw the last exception we got.
            ExecutionException ee = null;
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Iterator<? extends Callable<T>> it = tasks.iterator();

            // Start one task for sure; the rest incrementally
            futures.add(ecs.submit(it.next()));
            --ntasks;
            int active = 1;

            for (;;) {
            	// 循环会首先检查是否已经有执行完成的任务
                Future<T> f = ecs.poll();
                if (f == null) {
                    // 还有待执行的任务存在时一定会首先执行待完成任务
                    if (ntasks > 0) {
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    // 如果已经没有执行中的任务(到这里一定没有待执行任务了)则退出循环
                    else if (active == 0)
                        break;
                    // 超时等待处理--实际超时时间肯定会比传入的时间长，因为存在提交执行
                    // 任务等操作
                    else if (timed) {
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        if (f == null)
                            throw new TimeoutException();
                        nanos = deadline - System.nanoTime();
                    }
                    // 非超时等待则会阻塞
                    else
                        f = ecs.take();
                }
                // 如果有执行完成的任务，则尝试获取执行结果，
                // 获取失败会记录异常，然后继续执行其他任务或等待其他任务完成
                if (f != null) {
                    --active;
                    try {
                        return f.get();
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
            }

            if (ee == null)
                ee = new ExecutionException();
            throw ee;

        } finally {
            // 最后会取消未完成的任务，已完成的任务调用cancel方法不会产生任何效果
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
        }
    }
```

**invokeAll**

invokeAll方法会循环提交指定任务列表，并在提交完成后循环执行返回Future的get方法产生阻塞，以尽量保证方法返回前任务执行完成，最后未完成执行的任务会被取消。

```java
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            // 循环提交待执行任务
            for (Callable<T> t : tasks) {
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                execute(f);
            }
            // 循环获取执行结果，这里只是为了阻塞，取得的结果并不会被使用
            for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    try {
                        f.get(); // 阻塞保证任务执行完成才能返回
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t));

            final long deadline = System.nanoTime() + nanos;
            final int size = futures.size();

            // Interleave time checks and calls to execute in case
            // executor doesn't have any/much parallelism.
            // 循环提交待执行任务并检查超时时间
            for (int i = 0; i < size; i++) {
                execute((Runnable)futures.get(i));
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L)
                    return futures;
            }

            for (int i = 0; i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    if (nanos <= 0L)
                        return futures;
                    try {
                        f.get(nanos, TimeUnit.NANOSECONDS); // 超时阻塞尽量保证任务完成返回
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    } catch (TimeoutException toe) {
                        return futures;
                    }
                    nanos = deadline - System.nanoTime();
                }
            }
            done = true;
            return futures;
        } finally {
            // 等待超时后未执行完成的任务将会被取消
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }
```