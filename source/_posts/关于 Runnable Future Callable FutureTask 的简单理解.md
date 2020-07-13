---
title: 关于 Runnable Future Callable FutureTask 的简单理解
date: 2019-10-20 16:02:26
tags: [多线程, Future,FutureTask,Java]
---



### 1. Runnable 源代码

```
public interface Runnable {
    public abstract void run();
}

```

但是 Runnable 不允许声明受检异常以及定义返回值。不允许声明受检异常意味着必须手动的捕获并处理受检异常，这是十分麻烦的。

<!-- more -->

Executors 中存在可以将 Runnable 转换成 Callable 的方法。

```
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }

    public static Callable<Object> callable(Runnable task) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<Object>(task, null);
    }
```

在 Java 1.5 后引入了 Callable 和 Future 使得以上问题能够被优雅的解决。

### 2. Callable 源代码

```
public interface Callable<V> {
    V call() throws Exception;
}
```

可以在 Callable 的实现类中声明返回的数据类型，也可以抛出异常。

```
public class CallableTask implements Callable<String> {

    @Override
    public String call() throws Exception {
        String name;
        Thread.sleep(3000);
        name = "Mike";
        System.out.println("我是 CallableTask，执行 call 的线程为: " + Thread.currentThread().getName());
        return name;
    }
}
```
### 3. Future、FutureTask 相关


#### 3.1 Future 部分源码


```
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    /**
     * Returns {@code true} if this task was cancelled before it completed
     * normally.
     */
    boolean isCancelled();
    /**
     * Returns {@code true} if this task completed.
     */
    boolean isDone();

    /**
     * Waits if necessary for the computation to complete, and then
     * retrieves its result.
     * 在返回执行结果前，一直阻塞
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * Waits if necessary for at most the given time for the computation
     * to complete, and then retrieves its result, if available.
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
Runnable 和 Callable 一旦开启，就难以管理，而 Future 为线程池制定了一个可管理的任务标准，它提供了对 Runnable  和 Callable 任务的执行结果进行取消、查询是否完成、获取结果、设置结果的操作，分别对应于 cancel、isDone、get、set 函数，get 方法会阻塞，直到返回结果。


#### 3.2 FutureTask 部分源码


```
public class FutureTask<V> implements RunnableFuture<V> {
    ...
    ...
    ...
}


public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}

```

Future 只是定义了一些接口规范，而 FutureTask 则是它的实现类。

FutureTask 理解为 **可以被取消的任务**，它提供实现了 Future 的基本方法：启动、终止、查询是否执行完毕、返回执行结果。

可以通过执行 get() 获取执行结果，在任务执行结束之前，会一直阻塞，直到任务执行完成，获取结果。**阻塞的原因**为执行的循环 `for(;;)`, 只有在状态为结束后才会继续执行。


```
// FutureTask#get()
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}

private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        //最终返回状态，如果状态为完成，则执行 report(s)
        return state;
        
    }
}

public void run() {
    
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 执行 call 方法，最终执行相应的操作，置位状态
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```
s
Future 实现了 RunnableFuture，而RunnableFuture 实现了 Runnable 和 Future，所以它可以被当做 Runnable 被线程执行，也可以作为 Future 得到 Callable 的返回值。

#### 3.3 通过 Thread 执行 FutureTask 任务

```
FutureTask<Integer> futureTask
            = new FutureTask<>(() -> 1 + 2);
Thread thread = new Thread(futureTask);
thread.start();
System.out.println("result 2： " + futureTask.get().intValue());
```


#### 3.4 通过 ThreadPoolExecutor 执行 FutureTask 任务

通过线程池执行 FutureTask 任务：
```
FutureTask<Integer> futureTask
            = new FutureTask<>(() -> 1 + 2);
ExecutorService es =
        Executors.newCachedThreadPool();
// 提交 FutureTask
es.submit(futureTask);
// 获取任务执行结果
Integer result = futureTask.get();
System.out.println("result： " + result);
```

下面具体看一下 ExecutorService 为何物。


```
public interface ExecutorService extends Executor {
    // 提交 Callable 任务
<T> Future<T> submit(Callable<T> task);
// 提交 Runnable 任务，并声明返回值类型
<T> Future<T> submit(Runnable task, T result);
// 提交 Runnable 任务
Future<?> submit(Runnable task);
}
```

在 ExecutorService 中声明了 3 个 submit() 方法和一个定义的 Future 类用来支持获得任务执行的结果的需求。

这 3 个方法在 AbstractExecutorService 中具体实现。


```

public abstract class AbstractExecutorService implements ExecutorService {
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
}
```

最终都是通过 newTaskFor() 方法获得 FutureTask 对象：
```
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
}
```

而 execute 在不同的线程池中有不同的实现方式，具体可以查看线程池的相关源代码。


### 4.Runnable、Callable 、Future 、FutureTask 异同

Future Callable FutureTask(FutureTask 由于实现了 RunnableFuture  是可以使用在 Thread 的，但是运行效果和 Runnable 一样，没有什么意义) 只能运用到线程池中，而 Runnable 既能运行在 Thread 中，又能运用在线程池中。

--- 




**知识链接：**

[一个并发编程系列](https://www.cnblogs.com/dolphin0520/p/3949310.html)