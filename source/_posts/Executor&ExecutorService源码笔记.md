---
title: Executor&ExecutorService源码笔记
date: 2017-12-20 12:02:32
summary: 本篇对Executor及ExecutorService接口源码做了简单分析及整理。
tags: 多线程
---
## Executor接口
### 概要
> * Executor用于执行提交的任务，而这些任务所对应的类都应该实现了Runnable接口。利用Executor可以将提交任务的过程与具体执行任务的过程解耦。包括任务的具体执行机制，线程的使用细节，调度机制等等都可以依赖于Executor的具体实现。
> * Executor通常可以用来替代显示创建线程的操作，比如下面这个例子：
> ```java
>  // 显示创建线程
>  new Thread(new RunnableTask()).start();
>  // 通过Executor隐藏线程创建细节
>  Executor executor = someExecutor;
>  executor.execute(new RunnableTask());
> ```
> * Executor接口并没有强制要求执行的过程必须是异步完成的。因此，可以像下面的例子那样直接在execute方法调用线程上立即执行提交的任务。
> ```java
> class DirectExecutor implements Executor {
> 	public void execute(Runnable r) {
> 		r.run();
>  	}
> }
> ```
> * 当然，你还可以像下面这样为每一个任务创建一个新的线程。
> ```java
>  class ThreadPerTaskExecutor implements Executor {
>    public void execute(Runnable r) {
>      new Thread(r).start();
>    }
>  }
> ```
>
>  *注：以上例子摘自官方API文档
### 主要方法
Executor接口只定义了一个execute方法，提交的任务会在execute方法被调用后的某个时刻被执行，而具体的执行细节会在具体的实现类里得到体现。
```java
// 如果Executor拒绝任务的提交，则会抛出RejectedExecutionException
void execute(Runnable command);
```


## ExecutorService接口
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
需要注意的是：本方法没法保证很好地中止任务的执行，举例来说，通常改方法的实现Thread
