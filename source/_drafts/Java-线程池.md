---
title: Java 线程池
tags:
---

[java并发编程--Executor框架
](https://www.cnblogs.com/MOBIN/p/5436482.html)


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


FixedThreadPool：

    通过 Executors 的 newFixedThreadPool方法来创建，通过类名我们也可以得知 FixedThreadPool 为线程数量固定的线程池，除非关闭线程池，否则即使线程池中的线程处于闲置状态时，这些线程也不会被回收。