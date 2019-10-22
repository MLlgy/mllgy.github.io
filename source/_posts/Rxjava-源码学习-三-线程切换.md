---
title: 'Rxjava 源码学习(三):线程切换'
date: 2019-10-21 15:06:53
tags:
---


根据官方文档的翻译：
subscribeOn 操作符指定 Observable 将在哪个线程上开始操作，无论该运算符在运算符链中的哪个点被调用。 ObserveOn 操作符影响下游 Observer 的运行线程，因此，可以在 Observable 操作符链中的各个点多次调用 ObserveOn，以更改某些运算符的运行线程。

<!-- more -->

* subscribeOn： 指定 Observable 运行的线程，更近一步就是指定事件发射器发射动作的线程，但是 **只有第一个 subscribeOn 有效**。其实多个 subscribeOn 对整个事件流是有影响的，比如指定上游操作符对应的 Observer 的生成和订阅关系的建立所在的线程，但是这些都是发生在 Rxjava 库内部，不用也没有必要暴露给使用者，所以 subscribeOn 最终体现是：**指定事件发射器发射动作的线程**。
    
   
* observeOn：指定该操作符影响与下一个 observeOn 之间的 Observer 运行的线程。
  
![Rxjava 线程切换](/../images/2019_10_18_02.png)

为了明确 Rxjava 中相应的 Observer、Observable 运行在哪个线程，我们先看一个基本的关于 Thead 的知识点，帮助理解：

```
fun main(args: Array<String>) {
    thread(name = "Outer-Thread") {
        println("thread name is ${Thread.currentThread().name}")
        thread(name = "Inner-Thread") {
            println("thread name is ${Thread.currentThread().name}")
        }
    }
}
```

打印日志：
```
thread name is Outer-Thread
thread name is Inner-Thread
```


以上代码就是只有第一个 subscribeOn 生效和 subscribeOn 操作符不会影响 observeOn 操作符指定线程的原因。

其实这是很容易理解的，在这里说一下的原因是帮助理解。


### subscribeOn 操作符

同为 Rxjava 操作符，subscribeOn 的基本操作基本一致：生成相应的 Observer 、产生订阅关系，但是不同的是产生订阅关系所在的线程为 subscribeOn 指定的线程，下面看一下具体的源码：

```
@Override
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);
    observer.onSubscribe(parent);
    // scheduleDirect 操作会开启新的线程
    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}


final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;
    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }
    // 订阅关系产生在新的线程中
    @Override
    public void run() {
        source.subscribe(parent);
    }
}
```

其他操作符订阅关系发生的线程，以 map 操作符为例：
```
@Override
public void subscribeActual(Observer<? super U> t) {
    source.subscribe(new MapObserver<T, U>(t, function));
}
```


结合上面的流程图以及关于 Thread 的代码，这就是 subscribeOn 生效的原因。

具体说明一下，当 subscribeOn 指定线程后，那么就如流程图中所示，其后的操作都会在该线程中运行，最终事件发生器也会在该线程中发射新的事件。但是该 subscribeOn 操作符的上游再次使用了 subscribeOn ，那么事件发生器发布事件的线程最终由上游的 subscribeOn 指定的线程，具体原因为上文中提到的关于 Thread 的代码。


### observeOn 操作符

observeOn 操作符只能用来指定下游 Observer 的 onNext 等方法的线程，上面流程图可以非常形象的明确这一点。与 **subscribeOn 指定订阅关系发生的线程不同，observeOn 指定具体的动作的执行线程**，下面通过源码具体说明：

```
@Override
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker();
        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}

static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>{
@Override
public void onNext(T t) {
    if (done) {
        return;
    }
    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    schedule();
}

void schedule() {
    if (getAndIncrement() == 0) {
        worker.schedule(this);
    }
}

@Override
public void run() {
    if (outputFused) {
        drainFused();
    } else {
        drainNormal();
    }
}

// 最终发送的事件在指定的线程中执行
void drainNormal() {
    a.onNext(v);   
}
}
```

如果下游没有 observeOn 操作符，那么后续的 onNext 也同样在该线程中执行，同理，onError、onComplete 也会执行通过的操作，从而完成了 observeOn 操作符可以执行下游 Observer 的线程(具体来说是指定执行事件的线程)。




