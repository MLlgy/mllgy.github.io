---
title: 'Rxjava 源码学习(二):基本流程流程图'
date: 2019-10-18 15:32:54
tags: [Rxjava 源码分析]
---



在 [Rxjava 源码学习(一):基本流程分析](https://leegyplus.github.io/2019/10/18/Rxjava%20%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0(%E4%B8%80):%E5%9F%BA%E6%9C%AC%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/#more) 分析了基本流程，并且通过 Map 操作符一窥 Rxjava 操作符的特色，而本篇主体只有一张 Rxjava 流程图，但是这张流程图基本上可以概括 Rxjava 的框架，整个流程可以看做是一个 “横向 S” 模型。


该图共涉及 Rxjava 事件流的以下几个方面：
1. Observable 的创建
2. Observer 的创建、产生订阅关系
3. 订阅关系的传递
4. 取消订阅的流程


<!-- more -->
具体看图上的标记:

![](/../images/2019_10_18_01.png)


核心逻辑：

1. 开启订阅，即调用最后一个操作符生成的 Observerable(n) 和显式创建的 Observer(1)，并产生订阅关系 Observerable(n).subscribe(Observer(1))
2. 在 Observerable(n).subscribe(Observer(1)) 方法中将生成第 n 操作符生成的 Observer，即为 Observer(2),同时生成的 Observer(2) 也持有 Observer(1) 的引用，因为 Observerable(n) 持有 Observerable(n - 1) 的引用，那么此时会调用 Observerable(n - 1).subscribeActual(Observer(2))，向上游传递订阅关系；
3. 同时在 Observerable(n - 1).subscribeActual(Observer(2)) 也会调用 Observerable(n - 2).subscribe(Observer(3))，依次类推；

所以核心方法 subscribeActual 的主要工作为两个：
1. 生成本操作符下的 Observer
2. 调用 subscribe 方法，将上一个操作符生成的 Observerable 与本操作符生成的 Observer 产生订阅关系。

事件流的分发：

1. 按照以上流程重复操作，直到起始 Observerable 与对应的 Observer 产生订阅关系；
2. Observer 开始分发事件，由于订阅关系的存在，那么对应的 Observer 会收到对应事件；
3. Observer 调用自己的方法对事件进行处理，比如 map、apply等，同时由于上游 Observer 持有下游 Observer 的引用，那么将本 Observer 处理后的事件分发给下游的 Observer；
4. 以此类推，完成事件从上游到下游的传递；

### 1. 关于源码中的 upsteam 和 downstream

从图上可以看到，在最终订阅 Observer 之前，执行每一个操作符并不会同时生成相应的 Observable 和 Observer，以调用 subscribe 为分界线，将整个事件流分成两部分：
1. 调用 subscribe 之前，生成相应操作符的 Observable。
2. 调用 subscribe 之后，生成相应操作符的 Observer，并产生订阅关系。

需要注意的一点是在查看源码会看到 upstream、downstream，具体的 up 和 down 不是有相应对象的生成顺序决定的，而是有 Rxjava 相应操作符的调用先后决定。


### 2. 关于自定义 Observer 的 onSubscribe 方法的执行线程问题


Rxjava 中的 observerOn 和 subscribeOn 可以指定相应的 Observer 和 Observable 的运行线程，但是通过打印日志我们可以看到 onSubscribe 运行的线程并不是两个操作符指定的线程，而是代码执行的线程。


```
public void onSubscribe(Disposable d) {
    Log.e("TAG", "onSubscribe(): 所在线程为 " + Thread.currentThread().getName());
}
```
打印日志为：
onSubscribe(): 所在线程为 main


至于为什么会这样，可以看一下最上游的 Observable 的 subscribeActual 方法：

```
@Override
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);
    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}

@Override
public final void subscribe(Observer<? super T> observer) {
    subscribeActual(observer);
}
```
通过 [Rxjava 源码学习(一):基本流程分析](https://leegyplus.github.io/2019/10/18/Rxjava%20%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0(%E4%B8%80):%E5%9F%BA%E6%9C%AC%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/#more)  的分析知道，Rxjava 整个事件流的核心方法为 subscribeActual、subscribe，而两个方法均在代码调用的线程执行(这里是 main)，基于此 onSubscribe 方法就是在这里开始调用下游对象的 onSubscribe 方法，所以 onSubscribe 的执行线程也不会发生改变(这里是 main)。



### 3. 关于取消订阅关系

在日常开发中，我们会遇到类似这样的需求：当退出 Activity 时，需要取消正在执行的实现，此功能的实现就是通过取消订阅关系来实现。


情况一：

```
   Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("one");
                emitter.onNext("two");
                emitter.onNext("three");
            }
        }).subscribe(new Observer<String>() {
            Disposable mDisposable;
            @Override
            public void onSubscribe(Disposable d) {
                mDisposable = d;
            }

            @Override
            public void onNext(String s) {

                if(s == "two"){
                    mDisposable.dispose();
                }
            }
        });
```


情况二：

```
CompositeDisposable compositeDisposable = new CompositeDisposable();

private void test(){
   Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("one");
                emitter.onNext("two");
                emitter.onNext("three");
            }
        }).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                // 将订阅关系添加到 compositeDisposable
                compositeDisposable.add(d);
            }
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 解除所有的订阅关系
        compositeDisposable.clear();
    }
```

在自定义 Observer(最下游) 调用 Disposable#dispose():
```
@Override
public void dispose() {
    upstream.dispose();
}
```
 最终会通过事件流将取消订阅的动作传递到最上游：

 ```
// CreateEmitter#dispose
@Override
public void dispose() {
    DisposableHelper.dispose(this);
}
```

由于订阅关系取消，所以后续事件无法发布：

```
// CreateEmitter#onNext
@Override
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onN
        return;
    }
    // 取消订阅后 isDisposed 为 false
    if (!isDisposed()) {
        observer.onNext(t);
    }
}
CreateEmitter#isDisposed
@Override
public boolean isDisposed() {
    return DisposableHelper.isDisposed(get());
}
```


至此整个事件流被终止。