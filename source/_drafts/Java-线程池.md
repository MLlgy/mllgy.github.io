---
title: Java 线程池
tags:
---

[java并发编程--Executor框架
](https://www.cnblogs.com/MOBIN/p/5436482.html)


### Executor UML 图

![](/../images/2019_08_22_01.png)

### Executor


> An object that executes submitted {@link Runnable} tasks. This
interface provides a way of decoupling task submission from the
mechanics of how each task will be run, including details of thread
use, scheduling, etc.  An {@code Executor} is normally used
instead of explicitly creating threads. For example, rather than
invoking {@code new Thread(new RunnableTask()).start()} for each
of a set of tasks, you might use:

Executor 是可以执行提交任务单而对象。它解耦了任务的提交和任务的执行细节。Executor 可以代替创建新建线程，可以用：
```
Executor executor = anExecutor();
executor.execute(new RunnableTask1());
executor.execute(new RunnableTask2());
```
来代替 `new Thread(runnable).start();`。

但是 Executor 并不能保证任务能够异步执行，如下代码展示了任务在调用者的线程中执行的：


```
class DirectExecutor implements Executor {
  public void execute(Runnable r) {
    r.run();
  }
}}
```

在 [Main.Java](https://github.com/leeGYPlus/JavaCode/blob/master/src/excutors/Main.java) 中日志说明此处的 Executor 为同步执行。


但是更多的是任务在新的线程而不是调用者的线程中执行：

```
public class ThreadPerTaskExecutor implements Executor {

    @Override
    public void execute(Runnable runnable) {
        new Thread(runnable).start();
    }
}
```


Executor 的实现类限制了任务在何时以及怎样执行，同时也可以将一个线程池中的任务交给另外一个线程执行，比如 AsyncTask 中将串行线程池中任务交给并行线程池去执行。


#### ExecutorService

ExecutorService 继承了 Executor，是使用更加广泛的接口，提供了管理生命周期的方法以及跟踪一个或多个异步任务并返回 Future 的方法。 

ExecutorService 被关闭后就不会再执行新的任务，ExecutorService 提供了两个关闭 ExecutorService 的方法：

* void shutdown()：拒绝新的任务，但是之前的任务会被执行。
* List<Runnable> shutdownNow()：停止执行任何任务，返回等待执行的任务。

submit() 方法通过创建并返回可用于取消执行和/或等待完成的 Future来扩展基本方法 Executor#execute（Runnable）。


### AbstractExecutorService 

AbstractExecutorService 为 ExecutorService  的默认实现类，
Java 1.5 引入。


#### Executors 

Executors 提供了方便获得 ExecutorService 的工厂方法，如 FixedThreadPool、CachedThreadPool、ScheduledThreadPool、SingleThreadPool 等。

### ThreadPoolExcutor 参数含义

ThreadPoolExcutor 是线程池的真正实现，通过设置相应的参数来构建相应的线程池。ThreadPoolExcutor 构造函数如下：

```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,RejectedExecutionHandler handler) {
        
    }
```

corePoolSize:

    线程池的核心线程数，默认情形下核心先线程会一直存活，即使核心线程处于闲置状态，也会一直存活。

    如果设置 ThreadPoolExecutor 的属性  allowCoreThreadTimeOut 为 true，那么闲置的核心线程会在等待新任务到来前执行超时策略，而对于超时的时间由 keepAliveTime 决定。

maximumPoolSize：

    最大线程数，可允许创建的线程数，线程池中如果有这么多存活的线程，那么新到的任务会发生阻塞。

keepAliveTime：

    闲置的非核心线程在超过这个时长后就会被回收，如果 allowCoreThreadTimeOut 被设置为 true，那么核心线程在超时后也会被回收。

unit：
    keepAliveTime 的时间单位。

workQueue：

    线程池的任务队列，通过线程池的 execute 方法提交的 Runnable 对象被添加到这个队列中。

threadFactory：

    线程工厂，为线程池实现创建新的线程的功能，ThreadFactory 为一个接口，只有 newThread(Runnable runnable) 方法。

handler:

    定义处理被拒绝任务的策略，默认使用ThreadPoolExecutor.AbortPolicy,任务被拒绝时将抛出RejectExecutorException。

### ThreadPoolExecutor 执行步骤

ThreadPoolExecutor 执行任务时大致遵循如下规则：

    1. 线程池中的线程数量没有超过核心线程数，那么直接启动一个核心线程来执行任务。

    2. 如果线程池中的线程数量大于或等于核心线程数，那么将任务插入到任务队列中，排队等待被执行。
   
    3. 如果在步骤 2 中任务无法正常加入任务队列，绝大部分原因是因为任务队列已满，如果此时线程数量没有达到设置的线程池的最大数量，就会启动非核心线程来执行任务。
   
    4. 如果步骤 3 中线程数量已经达到线程池规定的最大值，那么就拒绝执行此任务，ThreadPoolExecutor 会调用 RejectedExecutionHandler 的 rejectedExecution 方法来通知调用者。



### 常见的四种线程池的创建

根据不同场景，为 ThreadPoolExecutor 配置不同参数来创建不同特性的线程池，常见的四种线程池有：FixedThreadPool、CachedThreadPool、ScheduledThreadPool、SingleThreadPool。


**FixedThreadPool**(固定数量的线程池)：

构建方法：

```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```


    通过 Executors 的 newFixedThreadPool方法来创建，通过类名我们也可以得知 FixedThreadPool 为线程数量固定的线程池，除非关闭线程池，否则即使线程池中的线程处于闲置状态时，这些线程也不会被回收。

    由构建方法可知，FixedThreadPool 只有核心线程，所以这些线程不会被回收，所以执行效率更高。当有新任务时，如果线程池中的线程尚未达到最大线程数，直接创建新的线程；如果所有的线程均处于活动状态，那么新任务将处于等待状态，直到线程池中有空闲线程为止，需要明确的是任务队列没有大小限制。


**CachedThreadPool**：

构建方法：
```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```


    由构建函数知 CachedThreadPool 没有核心线程，线程的数量的最大值为 Integer.MAX_VALUE。当新的任务来临时，如果线程池中存在处于空闲状态下的线程，那么会复用该线程来处理任务，否则会创建新的线程。线程池中的空闲线程如果超过 60s 会被回收。SynchronousQueue 为不可添加任务的队列，这就意味着任何新的任务都会被立即执行。




**ScheduledThreadPool**：

构建方法：

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
}
```

    限定核心线程数，不限制非核心线程数，空闲线程在  DEFAULT_KEEPALIVE_MILLIS 后被回收。ScheduledThreadPool这类线程池主要用于执行定时任务和具有固定周期的重复任务。


使用：

```
ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(4);
//2000ms后执行command
scheduledThreadPool.schedule(command,2000,TimeUnit.MILLISECONDS);
//延迟10ms后，每隔1000ms执行一次command
scheduledThreadPool.scheduleAtFixedRate(command,10,1000,TimeUnit.MILLISECONDS);
```

**SingleThreadPool**:

构建方法：
```
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

    由构建方法可知，SingleThreadPool 中只有一个核心线程，这意味着所有的任务都会被顺序执行。
    
    SingleThreadExecutor的意义在于统一所有的外界任务到一个线程中，这使得在这些任务之间不需要处理线程同步的问题。