---
title: 'Rxjava 源码学习(二):线程切换'
tags:
---

根据官方文档的翻译：
SubscribeOn运算符指定Observable将在哪个线程上开始操作，无论该运算符在运算符链中的哪个点被调用。 ObserveOn，而另一方面，影响线程可观察到的将使用下面看起来算哪里。因此，您可以在Observable运算符链中的各个点多次调用 ObserveOn，以更改某些运算符在哪些线程上进行操作。



* subscribeOn：
    1. 指定 Observable 运行的线程，但是只有第一个 subscribeOn 有效。
    2. 指定 **第一个 subscribeOn(如果有的话) 以上 Observer** 运行的线程
   
* observeOn：指定该操作符以下一个或多个 Observer 运行的线程
    1. 一个 Observer： 为该 observeOn 后的操作符后 observeOn
    2. 多个 Observer： 该observeOn 下的无其他 observeOn 操作符



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


其实这是很容易理解的，在这里说一下的原因是帮助理解。



首先看一下 subscribeOn 操作符：

既然 subscribeOn 也是 Rxjava 的操作符，那么在整个事件流中它也是其他操作符一样遵循 Rxjava 的 `横向 S` 模型，但是该操作符和其他操作符不同的是：**在该操作符的作用下生成的 Observer 和 该操作符生成的 Observable 产生订阅关系的线程由该操作符指定**。

有点绕？这句话中有三个主体：
1. 该操作符的作用下生成的 Observer(ObservableSubscribeOn)，记为 A1
2. 该操作符生成的 Observable(SubscribeOnObserver) ,记为 B1
3. 操作符指定的线程

下面看一下具体关系，一切都明了。


其中主体1 和 主体 2 都很好理解，具体可以看一下 `横向 S` 模型，下面主要看一下主体3：


subscribeOn 作用下生成的 Observable，其中 scheduler 为指定的线程。
```
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```

接下来就是重点了，该操作符的下一个操作符生成的 Observable 记为 A2,生成的 操作符记为 B2，在 A2 和 B2 产生订阅关系过程中，调用了 A1.subsribleActual(B2)，源码如下：

```
@Override
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);
    observer.onSubscribe(parent);
    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```

```
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;
    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }
    @Override
    public void run() {
        source.subscribe(parent);
    }
}
```






subscribeOn 能够影响所有 Observable 的原因是在该操作下，相应 Observerable 和 Observer 的订阅关系发生在指定的线程中。observeOn 只能影响下游 Observer 的原因是其 onNext 方法相关动作发生在执行的线程中。






---
基本流程

线程切换
订阅 
背压
为什么有人反对使用 Rxjava？